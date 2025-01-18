---
layout: post
title:  Evaluate your afterEvaluate
header: header-eng
mastodon: 109485837722240140
---

You've probably seen dozens of Stack Overflow answers to Gradle problems to
simply adding `afterEvaluate {}` to your build. Sometimes, this fixes your
apparent issue, you commit it and move along. I'm here to tell you that you
should only ever use `afterEvaluate {}` as the last resort as most likely you
are just signing yourself up for future headaches.

[According to Gradle docs `afterEvaluate`](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html#afterEvaluate-org.gradle.api.Action-)
"adds an action to call immediately after this project is evaluated. [...]
Actions passed to this method execute in the same order they were passed. [...]
If you call this method within an afterEvaluate action, the passed action
executes after all previously added afterEvaluate actions finish executing."
This works great if you only ever have a single caller to `afterEvaluate`. Sadly,
most project have many plugins and custom build logic. Since `afterEvaluate`
actions depend on when they are scheduled, that means it also depends on the
order in which plugins were applied. This puts you in a world of execution order
pain.

Let's look at a specific example of where you might see it used. Let's say you
have a task that wants to output something to project's build directory. You
might start with
```kotlin
myTask.configure {
  outputFile = File(project.buildDir, "myFile.txt")
}
```
Let's say you then apply a plugin that does `project.buildDir = ...`, so now
depending on your plugin application order you might get a stale value. Stack
Overflow jumps in to help here to put you to
```kotlin
afterEvaluate {
  myTask.configure {
    outputFile = File(project.buildDir, "myFile.txt")
  }
}
```
Ok, now we are good? What if that plugin then does `afterEvaluate { project.buildDir = ... }`.
and applies after your plugin. 😮‍💨

What is much better is to avoid using `afterEvaluate` in the first place and only
use the value when you need it, which is exactly why Gradle has properties.
```kotlin
myTask.configure {
  outputFile.set(project.layout.buildDirectory.file("myFile.txt"))
}
```

This is just a tiny example of many. Pretty much every use of afterEvaluate
is suspect, so you should evaluate if you should be using some more robust
alternative.
