---
layout: post
title:  Going Offline - Steps to a hermetic Gradle build
header: header-eng
mastodon: 115748798372064969
---

By default, Gradle is effectively useless on an offline machine. Even fetching Gradle distribution itself requires
network access, not to mention build dependencies, project dependencies, test tools and so on. This is generally not
a problem for most folks as you have the internet and Gradle fetches things for you as it performs work. However, if you
want to be able to work offline (or behind a corp or a state firewall) or you want hermetic builds for higher likelihood
of reproducible builds. This post will walk through some of the steps to get there.

**Gradle distribution**

Download `gradle-9.2.1-bin.zip` and check it into your repository. Then set `distributionUrl=gradle-9.2.1-bin.zip` in
`gradle/wrapper/gradle-wrapper.properties` so that Gradle wrapper uses the local file to extract the Gradle distribution.

**Build dependencies**

You will want to download the build dependencies your build need and store it in your repository keeping the Maven
directory structure, such as `repository/com/android/tools/build/gradle/8.14.0`. In androidx we created and maintain [`importMaven`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:development/importMaven/README.md)
tool that allows you to call `./importMaven.sh com.android.tools.build:gradle:8.14.0` and the fetches all the `.jar`,
`.pom`, `.module`, `.asc`, and other files for this library and all of its dependencies.

Then, you need to tell Gradle to use this directory by adding the following to your `settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories {
        maven(url = "./repository")
    }
}
```

**Project dependencies**

Similar to build dependencies, download what you need and put it in `repository/` in your git repository.

Then, you need the following in `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        maven(url = "./repository")
    }
}
```

**Dependency Signatures**

With [signature verification enabled]({% post_url 2024-12-12-Trust-But-Verify-Dependencies %}) you'll want to set
`<key-servers enabled="false">` in `gradle/verification-metadata.xml` like:

```xml
<verification-metadata>
   <configuration>
      <verify-signatures>true</verify-signatures>
      <keyring-format>armored</keyring-format>
      <key-servers enabled="false">
         <key-server uri="https://keyserver.ubuntu.com"/>
         <key-server uri="https://keys.openpgp.org"/>
      </key-servers>
   </configuration>
</verification-metadata>
 ```

Then, running `./gradlew myTask --export-keys` will the keys fetched from key servers in `gradle/verification-keyring.keys`.

**Kotlin Native toolchain**

Kotlin Gradle Plugin by default will go to `https://download.jetbrains.com/kotlin/native/` to download dependencies for
the Kotlin native compiler (it doesn't use Gradle to fetch it). To address this, create a script to fetch the
relevant dependencies ([example here](https://github.com/liutikas/kotlin-native-offline-dependencies/blob/main/prepare.sh))
to store them in `konan-zips/` in your git repository.

Add `kotlin.native.distribution.downloadFromMaven=false` to `gradle.properties` and configure your Gradle plugin to do:

```kotlin
val zipPath = "file:${project.rootDir}/konan-zips"
project.extensions.extraProperties["kotlin.native.distribution.baseDownloadUrl"] = zipPath
project.tasks.withType(KotlinNativeCompile::class.java).configureEach {
    it.compilerOptions.freeCompilerArgs.add(
        "-Xoverride-konan-properties=dependenciesUrl=$zipPath"
    )
}
project.tasks.withType(CInteropProcess::class.java).configureEach {
    it.settings.extraOpts +=
        listOf(
            "-Xoverride-konan-properties",
            "dependenciesUrl=$zipPath"
        )
}
```

**Robolectric**

To prevent Robolectric test runner from fetching system images from the network (it doesn't use Gradle to fetch it),
you can do the following:

```kotlin
private fun configureJvmTestTask(task: Test) {
        task.systemProperty("robolectric.offline", "true")
        task.systemProperty(
            "robolectric.dependency.dir",
            "robolectric-images",
        )
    }
```

**Android Lint Checks**

There are some Android Lint checks that try to go to the network to get the latest data. To disable these checks you
can do:

```kotlin
android {
    lint {
        disable += "KtxExtensionAvailable" + "GradleDependency"
    }
}
```

**Verification**

How do we know that we succeeded? The best way is to run your build in a VM or a docker container. That way you ensure
you are not hitting any caches.

You can now go offline right in time for the holidays!
