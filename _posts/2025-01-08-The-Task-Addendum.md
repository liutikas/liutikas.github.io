---
layout: post
title:  The Task Addendum - Add Outputs to A Task You Don't Own
header: header-eng
mastodon: 113796696840710051
---

The best way to extend a functionality of a Gradle task is by adding an input or an output.
Sadly, sometimes it is not a task you can modify easily, and thus you are stuck dealing
with what you've got.

Let's say I'm writing a JUnit test in a Gradle JVM project and I want my test to store some
additional data (e.g. a diff image between a reference image and a rendered screenshot). Extending
`org.gradle.api.tasks.testing.Test` can be non-trivial, especially when you are dealing with the
Kotlin Gradle Plugin or the Android Gradle Plugin. So what can we do?

Naive approach would be to pass a directory path to the task via environment variable.

```kotlin
tasks.named<Test>("test").configure {
    val testOutput = layout.buildDirectory.dir("testOutput")
    environment("TEST_OUTPUT" to testOutput.get().asFile.absolutePath)
}
```

then in your test class you do the following
```kotlin
val testOutput = File(System.getenv("TEST_OUTPUT"))
File(testOutput, "myFile.txt").writeText("Hello world!")
```

If you run this test, you will get `build/testOutput/myFile.txt` created as expected. Sadly,
we have a few issues at hand. First, if `build/testOutput` gets deleted, and you run the test
task again, the directory will not be recreated. That is because we haven't told Gradle this
directory is an output for this task.

```kotlin
tasks.named<Test>("test").configure {
    val testOutput = layout.buildDirectory.dir("testOutput")
    outputs.dir(testOutput)
    environment("TEST_OUTPUT" to testOutput.get().asFile.absolutePath)
}
```

After this change, if `build/testOutput` directory is deleted, then `Test` task will rerun
and the directory will be recreated! Yay!

You might think we are done, but sadly, there is another issue if you have a remote build cache
enabled ([and you should!]({% post_url 2022-07-13-Stop-Wasting-Time %})) you will likely not cache
hits. This will happen because you set `TEST_OUTPUT` environment variable to an absolute path
that will likely be different on different machines. Luckily there is a cheeky fix!

```kotlin
tasks.named<Test>("test").configure {
    val testOutput = layout.buildDirectory.dir("testOutput")
    outputs.dir(testOutput)
    doFirst {
        environment("TEST_OUTPUT" to testOutput.get().asFile.absolutePath)
    }
}
```

What we did here is to move environment variable setting to *after* the task inputs are captured
as `doFirst` actions runs during task execution phase. This means that Gradle will not see
`TEST_OUTPUT` and the absolute path as an input at all, but your test will see it. This is a
somewhat hacky and I've been told by Gradle developers that they will likely close this door down
eventually by locking task inputs after configuration is complete, but it still works, so...

Hopefully, this post gives you ideas of how you can add outputs to `Test` and other tasks.
You can also do similar things with `Task.inputs`, but if you do, remember to also [normalize your
input]({% post_url 2022-11-07-Just-Sensitive-Enough-Inputs %}) like
```kotlin
inputs.dir(testInput).withPathSensitivity(PathSensitivity.RELATIVE)
```

Happy tinkering!