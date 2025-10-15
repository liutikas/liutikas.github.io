---
layout: post
title:  Gradle Version Checking - Finding an API hidden deep in the basement
header: header-eng
---

Every Gradle Plugin has an implicit minimum version of Gradle that it supports. If you simply write a plugin, test it
on the versions you care about, but you do no additional work, your users will likely learn about the unsupported
version with a runtime crash of `ClassNotFoundException` when your plugin calls and API that is not available in the old
version of Gradle used by your new user.

Many plugins want to have a better message to the users, and [thus they add code along the lines of](https://github.com/search?q=GradleVersion.current&type=code):

```kotlin
if (GradleVersion.current() < GradleVersion.version(minimumGradleVersion)) {
    throw RuntimeException("You need to use at least version $minimumGradleVersion")
}
```

This is drastically better as the message is clear and actionable.

Luckily, there is an even cleaner way of achieving this with a minimal effort by using a Gradle API that is hidden deep
in the basement - [`GradlePluginApiVersion.GRADLE_PLUGIN_API_VERSION_ATTRIBUTE`](https://docs.gradle.org/nightly/javadoc/org/gradle/api/attributes/plugin/GradlePluginApiVersion.html#GRADLE_PLUGIN_API_VERSION_ATTRIBUTE).

In your Gradle plugin project, you can add:

```kotlin
listOf("runtimeElements", "apiElements").forEach { configurationName ->
    configurations.named(configurationName).configure {
        attributes {
            attribute(GradlePluginApiVersion.GRADLE_PLUGIN_API_VERSION_ATTRIBUTE, objects.named("8.5"))
        }
    }
}
```

This will update your plugin's `.module` metadata file to include `"org.gradle.plugin.api-version": "8.5"` attribute.
Then, without any additional work on you, if the user tries to use your plugin with an older Gradle version, they will get:

```
* What went wrong:
Could not resolve all artifacts for configuration 'classpath'.
> Could not resolve com.mygroup:my-plugins:1.2.3.
  Required by:
      unspecified:unspecified:unspecified > MySettingsPlugin:MySettingsPlugin.gradle.plugin:1.2.3
   > Plugin com.mygroup:my-plugins:1.2.3 requires at least Gradle 8.5. This build uses Gradle 8.4.
```

One might ask, wouldn't it be nice if Gradle automatically set this attribute? I agree, and we now have [an FR tracking that](https://github.com/gradle/gradle/issues/35327).

This same mechanism of `GradlePluginApiVersion` can also be used to serve different variants of your plugin for users
on a different version of Gradle. That way, you can adopt new Gradle APIs in a part of your plugin without
punishing your users on old Gradle versions. Kotlin Gradle Plugin already users this mechanism to split their plugin
based on user's Gradle version. You can take a look at [their `.module` file](https://repo1.maven.org/maven2/org/jetbrains/kotlin/kotlin-gradle-plugin/2.2.20/kotlin-gradle-plugin-2.2.20.module)
that will return a different `.jar` for Gradle `8.0`, `8.1`, `8.2`, `8.5`, `8.6`, `8.8`, `8.11`, and `8.13`+.
