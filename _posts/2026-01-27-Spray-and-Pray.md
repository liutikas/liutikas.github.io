---
layout: post
title:  Spray and Pray - Cost of Overly-Broad Spotless Targets
header: header-eng
---

In the post on the [unseasonable Gradle configuration time]({% post_url 2025-11-02-Unreasonable-Configuration %})
[Spotless was a culprit](https://github.com/diffplug/spotless/issues/2717) of a huge increase in configuration time
for [Androidify](https://github.com/android/androidify) project. The post suggested removing Spotless Gradle plugin
from the build as the only workaround for the configuration time cost. Turns out an issue rests in the following
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

From a quick glance it looks reasonable, but if you look a bit longer you will spot that with `target("**/*.kt")` we are
telling Spotless that *any* directory in our project could contain a `.kt` file, which effectively means Spotless has
to check every single directory in the project. Can we be more precise? Yes we can!

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

[This trivial looking change](https://github.com/android/androidify/pull/194) has huge impact on Gradle configuration
time - it drops it by 45%!

Without this change Gradle profiler benchmark of `./gradlew build --dry-run` had 30,102 ms mean with 562 ms
standard deviation, but after the change it decreased to 16,482 ms mean with 236 ms standard deviation.

A well configured project should spend around 100 ms per project in configuration, but here androidify configuration
takes about 1030 ms per project. We still have more work to do!
