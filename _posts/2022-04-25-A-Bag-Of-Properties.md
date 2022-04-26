---
layout: post
title: A Bag of Properties
header: header-eng
---

[Gradle properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties) is one of the many ways to pass information to Gradle. It gives you type-unsafe way to define `String` key-value pairs similar to [Java properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html).

# Setting properties

You can set Gradle properties in 4 places (the first one found in any of these locations wins):
1. command-line with `-Pcom.foo=bar`
1. `gradle.properties` in [`GRADLE_USER_HOME`](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_environment_variables) directory
1. `gradle.properties` in project root directory (sometimes it works in subproject directories, but it is not supported)
1. `gradle.properties` in Gradle installation directory (e.g. if your project ships a [custom Gradle distribution](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:custom_gradle_distribution))

You are likely going to see command-line and project root `gradle.properties` most often. However, `GRADLE_USER_HOME` (default is `~/.gradle`) can be handy to set secret tokens or signing keys.

Gradle has [dozens of built-in properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties) such as `org.gradle.caching=true` and `org.gradle.parallel=true`. I highly recommend bookmarking that page and checking out available values every once in a while, as there are some gems like `org.gradle.caching.debug=true` that helps you debug Gradle task caching ([oh so many caches!](/2022/04/19/Caches-Everywhere.html)) bugs.

Android Gradle Plugin (AGP) also has an assortment of Gradle properties, but sadly these are scattered throughout the documentation and are a bit hard to find. It is the best to look at [`BooleanOption.kt` in AGP source code](https://cs.android.com/android-studio/platform/tools/base/+/mirror-goog-studio-main:build-system/gradle-core/src/main/java/com/android/build/gradle/options/BooleanOption.kt). Here you can find things like `android.defaults.buildfeatures.aidl=false` (to disable [AIDL](https://developer.android.com/guide/components/aidl) if you don't use it in your build) or `android.uniquePackageNames=true` to enforce unique Android package names throughout your projects.

Many other plugins have their own Gradle properties, for inspiration you can take a peek at [androidx project gradle.properties](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:gradle.properties) and see if any of them would work for you.

# Reading properties

You can check if a given property is set via [`project.hasProperty("foo.bar")`](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#hasProperty-java.lang.String-) and read it through [project.getProperties()](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#getProperties--) or [project.property("foo.bar")](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#property-java.lang.String-). This mechanism allows your own build to have these values, which can be useful for things like `-Pci=true`.

One frustrating thing with this system is that it is not type safe. If you look at the methods linked above, they all return `Object` type and then you have to cast it to
something more useful. Folks at Slack have [built a version of type-safe properties](https://github.com/slackhq/slack-gradle-plugin/blob/main/slack-plugin/src/main/kotlin/slack/gradle/util/PropertyUtil.kt) that does this a bit better.

# These are not the properties you are looking for

Continuing the theme of overloaded terms - `properties` is no exception. Gradle also has:
- [dynamic project properties](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#:~:text=Dynamic%20Project%20Properties) - which are responsible for a lot of "magic" in your `build.gradle` files
- [`Property` class](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/Property.html) used in a configuration context.
- [`Lazy properties`](https://docs.gradle.org/current/userguide/lazy_configuration.html#lazy_properties) to describe a general category of inputs that are not computed at configuration time.
- `local.properties` - technically in the same category as all the `gradle.properties`, but used for non-versioning-system-safe properties such as a path to Android SDK
