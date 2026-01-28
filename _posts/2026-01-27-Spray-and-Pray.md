---
layout: post
title:  Spray and Pray - Cost of Overly-Broad Spotless Targets
header: header-eng
mastodon: 115970490537930025
---

In the post on the [unseasonable Gradle configuration time]({% post_url 2025-11-02-Unreasonable-Configuration %})
[Spotless was the culprit](https://github.com/diffplug/spotless/issues/2717) for the huge increase in the configuration
time for [Androidify](https://github.com/android/androidify) project. As a workaround that post suggested removing
Spotless Gradle plugin, but it turns out there is a better way. Our performance issue rests in the following
Spotless configuration:

```kotlin
spotless {
    kotlin {
        target("**/*.kt")
        targetExclude("**/build/**/*.kt")
    }
    kotlinGradle {
        target("**/*.kts")
        targetExclude("**/build/**/*.kts")
    }
    format("xml") {
        target("**/*.xml")
        targetExclude("**/build/**/*.xml")
    }
}
```

From a quick glance it looks reasonable, but if you look at it a bit longer you will spot that with `target("**/*.kt")`
we are telling Spotless that *any* directory in our project could contain a `.kt` file, which effectively means Spotless
has to check every single directory in the project. Can we be more precise? Yes we can!

```kotlin
spotless {
    kotlin {
        target("src/**/*.kt")
    }
    kotlinGradle {
        target("*.kts")
    }
    format("xml") {
        target("src/**/*.xml")
    }
}
```

Instead of guessing whether this change improves the situation it is best to use [`gradle-profiler`](https://github.com/gradle/gradle-profiler)
to validate the results. Based on benchmarks [this trivial looking change](https://github.com/android/androidify/pull/194)
has a huge impact on the Gradle configuration time dropping it by 67%!

Without this change Gradle profiler benchmark of `./gradlew build --dry-run` had 30,102 ms mean with 562 ms
standard deviation, but after the change it decreased to 10,076 ms mean with 478 ms standard deviation.

A well configured project should spend less than 100 ms per project in configuration. Even after this change androidify
configuration takes about 630 ms per project. We still have more work to do!
