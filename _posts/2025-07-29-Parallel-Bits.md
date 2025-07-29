---
layout: post
title:  Parallel Bits - Which Parts of Gradle Run in Parallel
header: header-eng
mastodon: 114937939372031455
---

By default, nearly the entire invocation of Gradle is done serially, but there are ways to make it partially parallel.

The list is ordered from the mostly safe to less stable options.

**Parallel task execution for tasks in separate projects**

Historically, enabled when using [Gradle Parallel Execution](https://docs.gradle.org/current/userguide/performance.html#sec:enable_parallel_execution)
mode with `org.gradle.parallel=true`, but now it is also achieved by [enabling Gradle configuration caching feature](https://docs.gradle.org/current/userguide/configuration_cache.html)
with `org.gradle.configuration-cache=true`. Configuration cache enables stricter rules in the build and therefore is
safer to use than the parallel execution mode.

**Parallel task execution for tasks within the same project**

Enabling [Gradle configuration caching feature](https://docs.gradle.org/current/userguide/configuration_cache.html)
with `org.gradle.configuration-cache=true`. *Note*, despite `org.gradle.parallel=true` sounding like it would do this,
it does not in fact enable this except for a narrow set of tasks that use [Gradle worker API](https://docs.gradle.org/current/userguide/worker_api.html)

**Parallel execution for workers within a single task**

Implement tasks using [Gradle worker API](https://docs.gradle.org/current/userguide/worker_api.html) and enable either
[Gradle configuration caching feature](https://docs.gradle.org/current/userguide/configuration_cache.html)
with `org.gradle.configuration-cache=true` or [Gradle Parallel Execution](https://docs.gradle.org/current/userguide/performance.html#sec:enable_parallel_execution)
mode with `org.gradle.parallel=true`.

**Parallel IDE model building during Gradle Sync**

Enabling [Gradle Parallel Execution](https://docs.gradle.org/current/userguide/performance.html#sec:enable_parallel_execution)
mode with `org.gradle.parallel=true` and using [Android Studio Electric Eel](https://android-developers.googleblog.com/2023/01/android-studio-electric-eel.html)
or newer.

**Parallel test execution within a JVM test task**

Setting the number of parallel forks via [Test.setMaxParallelForks](https://docs.gradle.org/nightly/javadoc/org/gradle/api/tasks/testing/Test.html#setMaxParallelForks(int)).

**Parallel Android Gradle Plugin R8 task execution**

Setting [`android.r8.maxWorkers`](https://issuetracker.google.com/issues/213907850)

**Parallel Configuration Cache loading**

Enabling [Gradle configuration caching feature](https://docs.gradle.org/current/userguide/configuration_cache.html)
with `org.gradle.configuration-cache=true` does this automatically as long as you use [Gradle 8.11](https://docs.gradle.org/8.11/release-notes.html#configuration-cache-improvements)
or newer.

**Parallel Configuration Cache storing**

Enabling [Gradle configuration cache parallel store feature](https://docs.gradle.org/current/userguide/configuration_cache.html#config_cache:usage:parallel)
with `org.gradle.configuration-cache.parallel=true` when using [Gradle 8.11](https://docs.gradle.org/8.11/release-notes.html#configuration-cache-improvements)
or newer. Partially unsafe as it requires build logic and all plugins to avoid reaching cross-projects while configuring
projects.

**`build.gradle(.kts)` compilation**

Enable [Gradle Isolated Projects feature](https://docs.gradle.org/current/userguide/isolated_projects.html) with
`org.gradle.unsafe.isolated-projects=true`. Quite unsafe as Isolated Projects is still incubating.

**Parallel configuration of projects**

Enable [Gradle Isolated Projects feature](https://docs.gradle.org/current/userguide/isolated_projects.html) with
`org.gradle.unsafe.isolated-projects=true`. Quite unsafe as Isolated Projects is still incubating.

I hope this helps you get a sense of the toggles you have to control the Gradle build parallelism.
