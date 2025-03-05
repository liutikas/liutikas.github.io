---
layout: post
title:  That's Gonna Leave A Mark - Gradle Plugin Markers
header: header-eng
mastodon: 114112468305378132
---

You've likely encountered plugin declarations like this:

```kotlin
plugins {
    id("com.android.library") version("8.9.0")
    id("maven-publish")
}
```

And you might have wondered how Gradle locates these plugin. This post aims to demystify that process.

The simpler aspect is that Gradle includes a set of plugins within its distribution, known as [core plugins](https://docs.gradle.org/current/userguide/plugin_reference.html#plugin_reference).
Examples include `id("java-library")`, `id("maven-publish")`, and `"id(signing")`. Since these plugins are part of
[the Gradle distribution]({% post_url 2024-12-18-What-The-Distribution %}), their JAR files are already present on your
machine by when you execute Gradle.

The more intricate part involves [community plugins](https://docs.gradle.org/current/userguide/plugin_basics.html#2_community_plugins)
that are hosted on Maven repositories. By default, Gradle searches for plugins exclusively on Gradle Plugin Portal.
However, you can extend this search by configuring additional repositories your in `settings.gradle.kts` file:

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

Plugins are essentially JVM libraries. To translate a plugin ID to its corresponding Maven coordinates, Gradle uses
[Plugin Marker Artifacts](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_markers). These act as
transparent redirects.

For instance, when you declare `id("com.android.library) version("8.9.0")`, Gradle attempts to retrieve
`com.android.library:com.android.library.gradle.plugin:8.9.0` from each repository specified in your
`settings.gradle.kts`. Ultimately, it fetches [the POM file](https://dl.google.com/android/maven2/com/android/library/com.android.library.gradle.plugin/8.9.0/com.android.library.gradle.plugin-8.9.0.pom),
which contains a dependency on the actual plugin implementation:

```xml
  <dependencies>
    <dependency>
      <groupId>com.android.tools.build</groupId>
      <artifactId>gradle</artifactId>
      <version>8.9.0</version>
    </dependency>
  </dependencies>
```

A significant advantage of Plugin Markers is that you relocate a plugin's implementation to a different Maven coordinate
without requiring any modifications from end-users.

If you are developing a [binary Gradle plugin](https://docs.gradle.org/current/userguide/implementing_gradle_plugins_binary.html)
and need to configure another plugin, such as `id("com.android.library)`, you can include its marker artifact as
a `compileOnly` dependency:

```kotlin
dependencies {
    compileOnly("com.android.library:com.android.library.gradle.plugin:8.9.0")
}
```

To streamline this process, you can create a helper function to generate the Maven coordinate from a plugin ID and
version:

```kotlin
fun idToMavenCoordinate(id: String, version: String): String {
    return "$id:$id.gradle.version:$version"
}
```
