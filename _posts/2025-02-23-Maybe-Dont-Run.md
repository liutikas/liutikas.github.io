---
layout: post
title:  Maybe Don't Run - Optional Tasks
header: header-eng
---

You might encounter situations where a Gradle task shouldn't run based on condition that's expensive to compute.
There are several ways to handle this, and I'll walk you through the options.

**1. Conditional task registration**

```kotlin
fun potentiallyExpensiveCall(): Boolean = TODO()

if (potentiallyExpensiveCall()) {
    tasks.register<MaybeImportantTask>("maybeImportant")
}
```

This approach prevents the task from being registered when the condition is `false`, thus avoiding its execution.
However, it has significant drawbacks:
- it will call eagerly call `potentiallyExpensiveCall` every time configuration is run regardless of whether the task is ultimately run.
This leads to unnecessary I/O or other costly operations 
- if configuration cache is enabled, Gradle will track the files read in `potentiallyExpensiveCall` and use them
as inputs to configuration. Changes to these files will invalidate the configuration cache even when running an unrelated
task.

**2. Using `Task.setEnabled`**

```kotlin
tasks.register<MaybeImportantTask>("maybeImportant") {
    enabled = potentiallyExpensiveCall()
}
```

This approach registers the task, but disables it so if `potentiallyExpensiveCall` returns `false`. This avoids the issue
of handling potentially non-existent task dependencies. However, it still suffers from the same downsides as the first
option.

**3. Using `Task.onlyIf`**

```kotlin
tasks.register<MaybeImportantTask>("maybeImportant") {
    onlyIf {
        potentiallyExpensiveCall()
    }
}
```

This is the preferred approach for expensive conditions. The task is registered, but `potentiallyExpensiveCall` is
only invoked during the task execution. This prevents performance degradation during configuration
and avoids unnecessary configuration cache invalidation.

**4. Using [`@SkipWhenEmpty`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/SkipWhenEmpty.html)
annotation**

The final approach is specifically useful when `potentiallyExpensiveCall` checks for the existence of some files.

```kotlin
abstract class MaybeImportantTask : DefaultTask() {
    @get:[InputFiles SkipWhenEmpty]
    abstract val inputFiles: ConfigurableFileCollection
    // ...
}
tasks.register<MaybeImportantTask>("maybeImportant") {
    inputFiles.from(fileCollectionProvider())
}
```

In this setup, the task is registered, but if `inputFiles` is empty, the task is skipped during execution. This is
particularly handy for scenarios like skipping a compression task when there are no input files to process.

*Important note*, first two options (conditional registration and `enabled = false`) are perfectly acceptable when
the condition check (`potentiallyExpensiveCall`) is inexpensive.