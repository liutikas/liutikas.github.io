---
layout: post
title: Caches Everywhere
header: header-eng
---

If you are working with a Gradle build you have likely heard term *cache* used many times. In this post I wanted to shed some light on the type of caches that are out there.

### GitHub Actions side

- [*GitHub Actions cache*](https://github.com/actions/cache) - a generic state store/restore system for any tool running on GitHub actions. Their documentation includes [an example of how to handle Gradle builders](https://github.com/actions/cache/blob/main/examples.md#java---gradle) (**note** don't use this for your Gradle builds, see next entry)

- *[Gradle build action](https://github.com/gradle/gradle-build-action) cache* - a state store/restore system for Gradle runs on GitHub actions, that works a lot better than the generic GitHub cache action. It is maintained by Gradle itself. You should be using this if you use GitHub Actions.

### Gradle build tool side

- [*Gradle configuration cache*](https://docs.gradle.org/current/userguide/configuration_cache.html) - Gradle feature to store/restore the state of the [*configuration phase*](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases) that you likely know from your terminal as:
```
<-------------> 0% CONFIGURING [4s]
```
Gradle serializes all the tasks with all the inputs that might invalidate how the task is set up during the first run, and for the later runs it reloads these from disk instead of rebuilding the whole task graph again, essentially making it so that `./gradlew test` on the second run only run the tests skipping the step of figuring out how to configure this task. This feature is currently (as of April 2022) incubating, but it works well and [should be adopted](https://docs.gradle.org/current/userguide/configuration_cache.html#config_cache:usage:enable).

- [*Gradle local build cache*](https://docs.gradle.org/current/userguide/build_cache.html#build_cache) - a mechanism to skip Gradle [task execution phase](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases) by storing/restoring outputs for a given input state to disk. For example, if you run a [cacheable task](https://docs.gradle.org/current/userguide/build_cache.html#sec:task_output_caching_details), detete the `build/` directory, and then run the task again, Gradle will skip running it and just directly restore the output from the first task. This is in particular helpful when switching between branches. This feature is stable, but **sadly** off by default, you [should be adopting it](https://docs.gradle.org/current/userguide/build_cache.html#sec:build_cache_enable)

- [*Gradle remote build cache*](https://docs.gradle.org/current/userguide/build_cache.html#sec:build_cache_configure_remote) - a feature equivallent to local build cache except using a remote server to store task inputs/outputs instead of local disk. This type of cache is often used by having Continious Integration (CI) builder pushing to this server, and all the local developers reading this cache. You can set this up using [Gradle Enterprise](https://gradle.com/gradle-enterprise-solutions/build-cache/), [Gradle Cache Node](https://docs.gradle.com/build-cache-node/), [AWS S3](https://github.com/myniva/gradle-s3-build-cache) or [GCP storage buckets](https://github.com/androidx/gcp-gradle-build-cache).

- [*GRADLE_USER_HOME*](https://docs.gradle.org/current/userguide/directory_layout.html#dir:gradle_user_home) - a directory (by default `~/.gradle`) that Gradle uses to store various non-project specific global state, including Gradle wrapper cache (extracted `gradle-X.X-bin.zip`) and artifact cache (to avoid redownloading maven dependencies)

- *Android Studio cache* - the infamous `Invalidate Caches / Restart` that solves all of your Studio problems. This is in fact does not touch any of the Gradle caches, instead [it clears various Intellij state like indexes](https://www.jetbrains.com/help/idea/invalidate-caches.html).
