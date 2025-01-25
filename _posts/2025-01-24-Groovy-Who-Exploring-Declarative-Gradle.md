---
layout: post
title:  Groovy Who? Exploring Declarative Gradle
header: header-eng
mastodon: 113886361338469722
---

You [probably spent a long time debating](https://github.com/gradle/gradle/issues/15886) on whether `build.gradle`
(Groovy) or `build.gradle.kts` (Kotlin Script) is more awesome. Lucky for you, we are about to have
[a third kid on the block](https://xkcd.com/927/) - `build.gradle.dcl` a.k.a. [Declarative Gradle](https://declarative.gradle.org/)
build file. I spent a week trying it out as part of my team hackathon to see how well this would work for a large build
like androidx and I would like to share some of my high-level findings.

**Disclaimer**: This is the experience as of declarative Gradle EAP 2, so things might change as features evolve.

Even before declarative Gradle, the AndroidX build has been slowly headed towards build files with the following shape:

```kotlin
plugins {
    id("AndroidXPlugin")
    id("com.android.library")
    id("kotlin-android")
}

dependencies {
    // lots of dependencies added by software developers 
}

androidx {
    // knobs that we want software developers to change
}
```

and the build logic is tucked away in an included Gradle plugin maintained and improved by a dedicated team.

The goal of the hackathon was to see if we could achieve a similar or better set up via `build.gradle.dcl`.

Did it work? Kind of! I was able to create software types like [`androidLibraryWithKotlin`](https://github.com/liutikas/androidx-declarative/blob/70bd39b7dcb531675fd2627d5af56904c63a7c58/core/core/build.gradle.dcl)
and [`jarLibraryWithKotlin`](https://github.com/liutikas/androidx-declarative/blob/70bd39b7dcb531675fd2627d5af56904c63a7c58/collection/collection/build.gradle.dcl)
that have dedicated single top level DSL exposing just the bits that make sense for an AndroidX library to vary on.
We get a structure like:
```kotlin
androidLibraryWithKotlin {
    dependencies {
        // lots of dependencies added by software developers 
    }
    deviceTest {
        dependencies {
            // lots of dependencies added by software developers 
        }
    }
    devicelessTest {
        robolectricEnabled = true
        dependencies {
            // lots of dependencies added by software developers 
        }
    }
    publishing {
        version = "1.0.2"
        group = ANDROIDX_COLLECTION
    }
    // other knobs that software developers can change
}
```

What are some highlights of declarative Gradle?
 
- Gradle Classic™ plugin skills transfer well to declarative, e.g. `@SoftwareType` is just a highly restricted Gradle
extension, you are still writing `Plugin<Project>` to wrap plugins to love/hate, etc.
- There is no way to write build logic in `build.gradle.dcl`. For a large build that's great as it gives build engineers
full control of what software engineer can do.
- IDE will only give you relevant autocomplete suggestions. You do not get 100k+ classes on the classpath like it was in
Groovy and Kotlin script. DSL scope is enforced, unlike non-declarative build file you could write `android { dependencies {`
putting dependencies inside the android block even if it does not belong here.
- It is trivial to parse (if you need to)

What are some lowlights?

- As of EAP 2, there is no way to apply more than one plugin, so you pretty much have to write your own ecosystem and
software type plugins, making it largely unusable for non-large builds due to complexity of migration and maintenance.
- Your software types are forced to be Property of primitives, String, or enum, so for example if you integrate with Kotlin
Gradle plugin, and you want to set `KotlinTarget`, you have to create your own mirror `KotlinTarget` enum that you
map to the real class.
- Many Gradle Classic™ Plugins are not designed to be easily set up in a lazy way, requiring duct-tape (`afterEvaluate`)
type workarounds.
- Mostly non-existent documentation, the only real way to learn is to dig through all the provided samples and try to
extract patterns how to use it.

Overall, I think this is looking very promising for large Gradle builds and I think androidx will likely adopt it as
the benefits seem to be worth it.
