---
layout: post
title:  Scope Reap - What are you calling?
header: header-eng
mastodon: 116452350627582061
---

Let's say we have `foo/bar/build.gradle` file with the following contents:

```groovy
plugins {
    alias(libs.plugins.androidLibrary)
}

android {
    compileSdk = 36
    namespace = "org.example.foo.bar"
    buildTypes {
        release {
            kotlin {
                compilerOptions { freeCompilerArgs = ["-Werror"]}
            }
        }
    }
}
```

A naive read of this would make it seem that this would cause `:foo:bar:compileReleaseKotlin` to turn Kotlin compiler
warnings into errors. The reality is not quite that simple and depends a lot on rest of the project set up.

**Using Android Gradle Plugin 9.0.0 or newer**

This code will make `:foo:bar:compileReleaseKotlin` fail on warnings, but it will unexpectedly fail any other Kotlin
compiler task in the project, for example `compileDebugKotlin` and `compileDebugUnitTestKotlin`. Why? Turns out that
`kotlin` is actually a top level extension, thus affected by the placement inside of `android { buildTypes { release {`.
It is equivalent to:

```groovy
plugins {
    alias(libs.plugins.androidLibrary)
}

android {
    compileSdk = 36
    namespace = "org.example.foo.bar"
    buildTypes {
        release {}
    }
}
kotlin {
    compilerOptions { freeCompilerArgs = ["-Werror"] }
}
```

In Groovy and by default in Kotlin script you can reach to anything from outer scopes. In Kotlin, you can prevent *some*
of these issues with [`@DslMarker`](https://kotlinlang.org/docs/type-safe-builders.html#scope-control-dslmarker), but
not this specific issue.

**Using Android Gradle Plugin 8.x**

This version of Android Gradle Plugin did not have built-in Kotlin, so generally Gradle should fail with:

```
A problem occurred evaluating project ':foo:bar'.
> Could not find method kotlin() for arguments [build_5ajgggwyonmf5wfwrp0p2ga61$_run_closure2@198010b] on project ':foo:bar' of type org.gradle.api.Project.
```

If we succeed, why? If *any* parent project such as root `:` or `:foo` applies Kotlin Gradle Plugin, you are configuring
Kotlin in those projects. Let's say `foo/build.gradle` has:

```groovy
plugins {
    alias(libs.plugins.kotlinJvm)
}
```

That means `-Werror` is actually will make `:foo:compileKotlin` and other Kotlin compile tasks fail on warnings.

Gradle has wired up Groovy dynamic invocation to look for `kotlin` in the following places:
- Plugin extensions named `kotlin` in `:foo:bar`
- Values in [`ExtraPropertiesExtension`](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html) in `:foo:bar` (e.g. `ext.kotlin = ...`)
- Gradle property named `kotlin` in `:foo:bar`

If not found, then repeats that project for each parent project, and only fail it is not found anywhere up the search
stack.

To make things extra confusing, Gradle properties are also layered. If you have `gradle.properties` with:
```properties
propA=root-propA
propB=root-propB
```

and you have `foo/gradle.properties` with:
```properties
propB=foo-propB
propC=foo-propC
```

And then in `foo/bar/build.gradle` has `println("$propA - $propB - $propC")` will print out:

```
root-propA - foo-propB - foo-propC
```

Scope? Scope! Cry.

This is the reason why Gradle fails when using `org.gradle.unsafe.isolated-projects=true` and you make a call
`Project.getProperty("missingProperty")`.
