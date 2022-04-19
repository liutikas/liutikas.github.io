---
layout: page
title: Gradle repositories {} Galore
permalink: /blog/1
header: header-eng
---

Gradle can be a really intimidating build tool to get started on. One of the concepts that baffled me in the beginning was `repositories {}` blocks throughout a Gradle project that contain a list of usually maven repositories used by Gradle to download artifacts. I could not make heads or tails of it, so I decided to share about it and helpfully help you understand it better.

First thing to be aware of that there are generally two types of repositories lists in Gradle: **plugin** vs **build**. You can think of these of *“how to build things”* vs *“what to build against”*. Plugin repositories are used to find where to find (surprise surprise…) Gradle plugins such as `spotless` or `kotlin`. Whereas build repositories are used to find what you want to use for your libraries, such as dependencies on `kotlin-stdlib` or `guava`.

Now that we have these two different definitions — the recommended way to set these are in your `settings.gradle.kts` with:

```
// plugin repositories
pluginManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
        google()
    }
}
// build repositories
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        google()
    }
}
```

`pluginManagement {}` block will be used to resolve entries like

```
plugins {
    id("com.android.application") version "7.2.0" apply false
}
```

and `dependencyResolutionManagement {}` block is used to resolve entries like
```
dependencies {
    implementation("com.google.guava:guava:31.1-jre")
}
```

However, given the title of this article you probably have already realized that these are not the only places where you can set repositories. You can head over to the [sample GitHub repo](https://github.com/liutikas/gradle-repositories-galore/) to look at [settings.gradle.kts](https://github.com/liutikas/gradle-repositories-galore/blob/main/settings.gradle.kts) and [build.gradle.kts](https://github.com/liutikas/gradle-repositories-galore/blob/main/build.gradle.kts) to see other places where you can use `repositories {}` and why you might want to do so.
