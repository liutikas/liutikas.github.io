---
layout: post
title:  Gradle Tower of Babel - Building on Top Of Other Plugins
header: header-eng
---

Gradle provides almost no guidance on how to build a Gradle plugin that builds on top of a different Gradle plugin.
In this post I want to share a sliver of details how to do it. Let's say you want to ship a plugin that adds a feature
on top of Android Gradle Plugin (AGP). A naive approach to it would be to add
```kotlin
dependencies {
    implementation("com.android.tools.build:gradle-api:8.0.0")
}
```
to your `build.gradle.kts` and start using it in your plugin code. This will work in the most basic cases, but sadly,
due to how build classloaders work you might accidentally force the user of your plugin into AGP `8.0.0` version even
though they want a newer version, e.g. if your plugin is added in `buildSrc` but AGP is applied at the project level,
the latter version will be partially overridden by your plugin dependency.

The fix for it is to instead use `compileOnly`
```kotlin
dependencies {
    compileOnly("com.android.tools.build:gradle-api:8.0.0")
}
```
which will avoid this unexpected AGP version override. Note, you should do that for *all* plugin dependencies and I
have [created a feature request for Gradle](https://github.com/gradle/gradle/issues/34898) to catch this.

Sadly, this change will likely break your plugin tests if you followed Gradle's documentation and use
[`GradleRunner.withPluginClasspath`](https://docs.gradle.org/nightly/javadoc/org/gradle/testkit/runner/GradleRunner.html#withPluginClasspath())
despite your test project specifying
```kotlin
plugins {
    id("com.android.application") version "8.0.0"
    id("net.liutikas.fancyPlugin")
}
```

with your test failing with something along the lines of
```text
* What went wrong:
An exception occurred applying plugin request [id: 'net.liutikas.fancyPlugin', version: '1.2.3']
> Failed to apply plugin 'net.liutikas.fancyPlugin'.
   > Could not create plugin of type 'FancyPlugin'.
      > Could not generate a decorated class for type FancyPlugin.
         > com/android/build/api/variant/ApplicationVariant
```

This happens because `withPluginClasspath` forces Gradle to behave in a way that is not at all reflective of how it
would when your user uses your plugin, instead if creates an "isolated" build classpath for which line
`id("com.android.application") version "8.0.0"` has no effect and thus your test results into a crash due to a missing
class. `withPluginClasspath` should pretty much never be used and `androidx.lint:lint-gradle` already flags that usage.

To make it work like you expect we have to do a delicate dance. First, in `build.gradle.kts` you need:

```kotlin
val repo = layout.buildDirectory.dir("repo")
tasks.withType<Test>().configureEach {
    // Make sure that build/repo is created and that it is used as input for the test task.
    dependsOn("publish")
    inputs.files(
        repo.map {
            // Exclude maven-metadata.xml as they contain timestamps but have no effect on the test outcomes
            it.asFileTree.matching { exclude("**/maven-metadata.xml*") }
        }
    ).withPathSensitivity(PathSensitivity.RELATIVE).withPropertyName("repo")
    systemProperties["plugin_version"] = project.version // so we can use the value in the test
    doFirst {
        // Inside doFirst to make sure that absolute path is not considered to be input to the task
        systemProperties["repo_path"] = repo.get().asFile.absolutePath // so we can use the value in the test
    }
}

publishing {
    repositories { maven { url = uri(repo) } }
    // other publishing set up
}
```

Then on the test side, you want to do the following:

```kotlin
@get:Rule val tempDirectory: TemporaryFolder = TemporaryFolder()
@Test
fun basic() {
    val projectDir = tempDirectory.newFolder("basic")
    File(projectDir, "build.gradle").writeText(
        """
        plugins {
            id("com.android.application") version "8.14.3" // can be any version
            id("net.liutikas.fancyPlugin") version "${System.getProperty("plugin_version")}"
        }
        repositories {
            google()
            mavenCentral()
        }
        android {
            compileSdk = 36
            namespace = "com.example.app"
        }
    """.trimIndent()
    )
    File(projectDir, "gradle.properties").writeText("android.useAndroidX=true")
    File(projectDir, "settings.gradle").writeText(
        """
        pluginManagement {
            repositories {
                maven {
                     // We are specifying the path to the local repo with our plugin
                     url = uri("${System.getProperty("repo_path")}")
                }
                google()
                mavenCentral()
            }
        }
        """.trimIndent()
    )
    val result = GradleRunner.create()
        .withProjectDir(projectDir)
        .withArguments("myFancyTask", "-s")
        .build()
    // Assert things on the result
}
```

You might say "oh my this is a lot of boilerplate" - I agree with you, hence I created [a feature request for Gradle](https://github.com/gradle/gradle/issues/34870)
to make this a lot simpler.

Happy coding!
