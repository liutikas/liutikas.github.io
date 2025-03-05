---
layout: post
title:  That's Gonna Leave A Mark - Gradle Plugin Markers
header: header-eng
---

You might have seen

```kotlin
plugins {
    id("com.android.library") version("8.9.0")
    id("maven-publish")
}
```

and wondered how does Gradle know where to find these plugin. I hope this post can help you answer that.

The easy part of the answer is that Gradle ships a number of plugins inside its distribution. Gradle calls them
[core plugins](https://docs.gradle.org/current/userguide/plugin_reference.html#plugin_reference) and examples of these
are `id("java-library")`, `id("maven-publish")`, and `"id(signing")`. As these plugins are part of [the distribution
Gradle]({% post_url 2024-12-18-What-The-Distribution %}) their jars already on your machine by the time you run Gradle.

The trickier bit is what happens to [community plugins](https://docs.gradle.org/current/userguide/plugin_basics.html#2_community_plugins)
that ship on a Maven server somewhere. By default, Gradle will only search for plugins on Gradle Plugin Portal, but you
can customize it in `settings.gradle.kts` using:

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

Plugins are just JVM libraries. To get from a plugin ID to the Maven coordinates for that plugin Gradle uses
[Plugin Marker Artifacts](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_markers). You can think of
them as a transparent redirect. For `id("com.android.library) version("8.9.0")` Gradle tries to fetch
`com.android.library:com.android.library.gradle.plugin:8.9.0` in every repository
you declared in your `settings.gradle.kts`. Eventually, it fetches [the pom file](https://dl.google.com/android/maven2/com/android/library/com.android.library.gradle.plugin/8.9.0/com.android.library.gradle.plugin-8.9.0.pom)
that has a dependency to the implementation of this plugin

```xml
  <dependencies>
    <dependency>
      <groupId>com.android.tools.build</groupId>
      <artifactId>gradle</artifactId>
      <version>8.9.0</version>
    </dependency>
  </dependencies>
```

Nice thing about the Plugin Markers is that you can move the implementation of a plugin to a completely different Maven
coordinate and your end users have to make no changes.

If you are building a [binary Gradle plugin](https://docs.gradle.org/current/userguide/implementing_gradle_plugins_binary.html)
for your build logic, and you want to configure another plugin, for example `id("com.android.library)`, you now know
you can add
```kotlin
dependencies {
    compileOnly("com.android.library:com.android.library.gradle.plugin:8.9.0")
}
```

If you are trying to codify it, you can use
```kotlin
fun idToMavenCoordinate(id: String, version: String): String {
    return "$id:$id.gradle.version:$version"
}
```
