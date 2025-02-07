---
layout: post
title:  Stifle Hungry Tasks using BuildService
header: header-eng
mastodon: 113959816328576682
---

On a reasonably sized project that follows [Gradle's best practices](https://github.com/liutikas/gradle-best-practices)
you might find your machine overloaded due to excessive parallelism. This is especially true for memory-intensive tasks
or tasks that internally utilize threads or coroutines. In the AndroidX project, we encountered this issue with static
analysis tasks using PSI/UAST. Running these tasks in parallel led to high memory usage, triggering constant garbage collection,
hence slowing down the entire build process to a crawl.

Fortunately, Gradle's [shared build services](https://docs.gradle.org/current/userguide/build_services.html) offer a
solution. However, we'll use them in a slightly unconventional way.

```kotlin
abstract class LimitingBuildService
    : BuildService<BuildServiceParameters.None> {
    companion object {
        const val KEY = "LimitingBuildService"
    }
}

val limitingService = gradle.sharedServices.registerIfAbsent(
    LimitingBuildService.KEY, LimitingBuildService::class.java
) {
    // Insert any fancy logic, such as "heapSize / expectedTaskMemUsage"
    maxParallelUsages.set(2)
}

abstract class MyHeavyTask : DefaultTask() {
    // Note, we have Gradle injection magic that will find the right
    // service by name and cast it to the type you expect
    @get:ServiceReference(LimitingBuildService.KEY)
    abstract val limitingService: Property<LimitingBuildService>

    @TaskAction
    fun doWork() {
        println("$name BEGINS")
        Thread.sleep(2000)
        println("$name ENDS")
    }
}

val allHeavyTasks = tasks.register("allHeavyTasks")
for (i in 0 .. 10) {
    val heavyTask = tasks.register<MyHeavyTask>("myHeavyTask$i")
    allHeavyTasks.configure { dependsOn(heavyTask) }
}
```

Here, the `BuildService` isn't used for actual work. Instead, it acts as a Gradle-managed lock, throttling the number of
concurrently running tasks. In our case, we guesstimated that each Android Lint task used about 512MB, allowing us to
determine the appropriate number of concurrent tasks.

The beauty of this approach is that it can be applied even to tasks that you do not own, like Android Lint analysis tasks:

```kotlin

tasks.withType(AndroidLintAnalysisTask::class.java).configureEach {
    usesService(limitingService)
}
```

Fortunately, Android Lint team adopted this approach and [implemented a much more sophisticated calculation](https://cs.android.com/android-studio/platform/tools/base/+/mirror-goog-studio-main:build-system/gradle-core/src/main/java/com/android/build/gradle/internal/services/LintParallelBuildService.kt)
in their own build service. This has effectively resolved the excessive garbage collection issues previously caused by Lint.
