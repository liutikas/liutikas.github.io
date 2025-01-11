---
layout: post
title:  Conservative Librarian - Hold Back For Your Users To Flourish
header: header-eng
---

There are a number of things that make software libraries easy to use. Giving library users the flexibility to choose
the tools and the versions of these tools is one critical piece of a great library experience. For a JVM library that
might be the version of the JDK needed to use this library, which is based on the Java bytecode level used for the classes
inside the `.jar`. If the library maintainer upgrades to the latest JDK (and you should, get all the compiler fixes
and performance improvements), by default Gradle will produce `.jar` files that use the latest bytecode level forcing
the library users to use the same version of the JDK. Upgrading JDK can be a really painful experience, so forcing is
not a great experience. Luckily for us JVM system has a concept of target Java version which allows you to use a newer
JDK to produce a `.jar` that contains only lower version bytecode. In Gradle, that would be:

```kotlin
plugins {
    id("java-library")
}
tasks.withType<JavaCompile>().configureEach {
    options.release.set(8)
    sourceCompatibility = "1.8"
    targetCompatibility = "1.8"
}
```

If your JVM project is using Kotlin Gradle Plugin (KGP), you also need to set the following:
```kotlin
tasks.withType<KotlinJvmCompile>().configureEach {
    compilerOptions.jvmTarget.set(JvmTarget.JVM_1_8)
    // Needed due to https://youtrack.jetbrains.com/issue/KT-49746
    compilerOptions.freeCompilerArgs.add("-Xjdk-release=1.8")
}
```

Doing this will allow your library consumers to use JDK 8 or newer.

Note: Gradle also has a feature called toolchains, and [even tells you to use it for targeting Java versions](https://docs.gradle.org/current/userguide/building_java_projects.html#sec:java_cross_compilation).
*Don't use it*, [see this post on why that is the case](https://jakewharton.com/gradle-toolchains-are-rarely-a-good-idea/).

JVM libraries that use Kotlin have an additional consideration - the Kotlin language level. By default, the version
of the KGP will determine the artifacts' Kotlin target language. If you upgrade your KGP to the latest (and you should
for the same reasons as JDK), you force library users to upgrade as well. Similar to Java bytecode level, you can control this.

```kotlin
kotlin {
    // makes sure kotlin-stdlib is at 1.9.0, as we stdlib also uses Kotlin language features
    coreLibrariesVersion = "1.9.0"
    // makes sure .class files only use Kotlin 1.9 language features
    compilerOptions.languageVersion.set(KotlinVersion.KOTLIN_1_9)
}
```

Doing this allow your library consumers to use Kotlin `1.9.0` or newer, even if the library is compiling with KGP `2.1.0`.
[KGP supports producing binaries that go back as far as 3 minor versions](https://kotlinlang.org/docs/kotlin-evolution-principles.html#evolving-the-binary-format)

If you own a Kotlin Multiplatform library, sadly, `.klib` artifacts [do not have such forward compatibility guarantees](https://kotlinlang.org/docs/kotlin-evolution-principles.html#kotlin-klib-binaries),
and thus, `.klib` artifacts produced by [KGP `2.1.0` cannot be consumed by library users that use KGP `2.0.0`](https://youtrack.jetbrains.com/issue/KT-68792/Bump-KLIB-ABI-version-in-2.1)
even if you set the target language version above.
