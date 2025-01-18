---
layout: post
title:  DRY - Gradle Configuration Values
header: header-eng
mastodon: 113387520995599741
---

As you work on your Gradle builds you might find a need to centralize your build configuration, for example you might
want to have a default `minSdk` for your projects that apply `com.android.library` plugin. The legacy way of
approaching this used to be adding build logic to the root `build.gradle.kts` along the lines of:

```kotlin
allprojects { project ->
    if (isAndroidLibrary()) configureAndroidLibrary(minSdk = 21)
}
```

This has several shortcomings, where one of the biggest ones is that it is not compatible with
[Gradle Isolated Projects](https://docs.gradle.org/current/userguide/isolated_projects.html) as you are forcing a
specific order of project evaluation.

Another way to do this is to have an [included build](https://docs.gradle.org/current/userguide/composite_builds.html#included_plugin_builds)
that has a custom Gradle plugin with the value hard-coded in code:

```kotlin
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        configureAndroidLibrary(minSdk = 21)
    }
}
```

and in `build.gradle.kts` files use:
```kotlin
plugins {
    id("myPlugin")
}
```

This works fairly well. In androidx project we have used this mechanism for years allowing us to keep hundreds of
our projects consistent. However, this way makes mixes declarative values with build logic making it harder for a
newcomer to a project to know where to change the value.

A much better ergonomics would be to be able to set these values in `settings.gradle.kts` in a declarative way
and get that picked up by all the relevant Project plugins. We have a way to do just that!

```kotlin
pluginManagement { includeBuild("build-logic") }
plugins {
    id("mySettingsPlugin")
}
myDefaults {
    minSdk.set(21)
}
```

This is implemented via a custom Gradle Settings plugin:
```kotlin
class MySettingsPlugin : Plugin<Settings> {
    override fun apply(settings: Settings) {
        val mySettings = settings.extensions.create("myDefaults", MySettingsExtension::class.java)
        settings.gradle.beforeProject { project ->
            project.extensions.getByType(ExtraPropertiesExtension::class.java).set("myDefaults", mySettings)
        }
    }
}
abstract class MySettingsExtension {
    abstract val minSdk: Property<Int>
}
```

Then this is read by your custom Gradle Project plugin:
```kotlin
class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val myDefaults = project.extensions.getByType(ExtraPropertiesExtension::class.java).get("myDefaults")
                as? MySettingsExtension ?: throw GradleException("Settings extension type mismatch")
        configureAndroidLibrary(minSdk = myDefaults.minSdk)
    }
}
```

This lets you keep your `settings.gradle.kts` nice and tidy! You can take a look at [an end-to-end example repository
on GitHub](https://github.com/liutikas/gradle-share-configuration).

Note, if you only need to set your `minSdk` default, Android Gradle Plugin team has in fact done all the work already
by creating the [`com.android.settings` settings plugin](https://developer.android.com/reference/tools/gradle-api/8.7/com/android/build/api/dsl/SettingsExtension#minSdk()),
so you can use that directly if you do not want to create your own.
