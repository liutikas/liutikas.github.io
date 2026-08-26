---
layout: post
title: Configuration Bomb for Eager Task Realization
header: header-eng
mastodon: 117158173511738128
---

Gradle has been pushing developers to migrate from the eager `tasks.create` to the lazy `tasks.register`. The former
does the work on task realization despite if that given task will be run, but the latter only does it when the task
will actually be needed. You might think that this optimization cannot possibly be that important, but you enter the
realm of 10k, 100k, or 1M of tasks these costs add up really quickly. Therefore, you should really take on this
migration and file bugs to any Gradle plugins that are still using the legacy eager way.

Sadly, like with many things with Gradle, there are footguns. Specifically, it is extremely easy to accidentally undo
the laziness of `tasks.register`. For example, call to any Gradle API that returns the task itself: `tasks.getByName`,
`tasks.withType` (without `.configureEach {}`) and Gradle has [a whole page on what to avoid](https://docs.gradle.org/current/userguide/lazy_configuration.html).
Another example is a lot more subtle, as it due to the fact that `org.gradle.api.tasks.TaskContainer` implements
`java.util.Collection`. Any use of `kotlin-stdlib` collection methods such as `tasks.map {}` or `tasks.all {}` will
eagerly realize every one of the tasks undoing all of your hard work.

Luckily, there is a very easy way to ensure this regression doesn't sneak into your build - a configuration bomb!

```kotlin
tasks.register("configurationBomb") {
  throw Exception("This task should never be configured. If this is called some code in the build used a non-lazy API.")
}
```

You effectively create a task in each project that should never be configured and if that ever changes, your build will
fail with the stacktrace to the culprit 💣.

If you want something out of the box, you can pick up [Eli Graber's Gradle Plugin](https://github.com/eygraber/gradle-config-bomb-plugin).
