---
layout: post
title:  Schrödinger's Gradle Inputs
header: header-eng
---

I ran into an unexpected Gradle behavior while debugging Gradle builds
not restoring from a remote cache as expected. This post explains how I got to
this observation, how this behavior can be used for a legitimate use-case, and
why you should probably avoid it anyway.

I started with the fact that local builds would always have a remote cache miss
for `KotlinCompile` tasks that the CI just built. I used Gradle build scans to
compare what was different between a local run and CI run. Scans told me
that `filteredArgumentsMap` values were different. To `build.gradle` I added
```groovy
tasks.withType(KotlinCompile).configureEach {
    it.doFirst {
        println(it.filteredArgumentsMap)
    }
}
```
and that let me find that in the CI we had `allWarningsAsErrors:true`, but this
entry was missing for the local builds. Turns out that due to historical reasons
wanted to prevent new warnings, white at the same time did not want local devs to have
failing builds due to an unused variable, and thus we added this flag just to
the CI builds. Luckily, since then, we added a [build log simplifier](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:development/build_log_simplifier/)
that catches any new warnings (not just from `KotlinCompile`), so we could
[change the default to CI to be `allWarningsAsErrors:false`](https://android-review.googlesource.com/c/platform/frameworks/support/+/2121872).
We started getting hits from the remote caches!

You might say _"Wait a minute! I was promised an unexpected behavior"_, and to be
honest I also thought I was done. Sadly, I got a report from
a co-worker that their project using [Compose](https://developer.android.com/jetpack/compose)
was also only getting remote cache misses for `KotlinCompile`. I jumped back
and did the same steps as before.

![Screenshot of build scan comparison page](/assets/2022-06-21-compare.png){: width="600" }

Once again I printed `filteredArgumentsMap` and this time I found
```text
freeArgs:[-Xplugin=/usr/local/.../compiler-1.1.1.jar, ...
```

I filed [a bug to Android Gradle Plugin (AGP) to not put
absolute paths](https://issuetracker.google.com/236707182), [tweeted about it](https://twitter.com/_aurimas/status/1539277224326463494),
and I thought was done.

_Narator's voice: he wasn't done_

A few minutes later, I got a ping from a co-worker on the AGP team telling me
that I was wrong and that remote cache does work for vanilla AGP projects, likely
pointing that AndroidX build logic is at fault. I was not convinced, I was seeing
absolute paths in `filteredArgumentsMap`. Then I learned that AGP is using
a `doFirst { }` lambda to modify `filteredArgumentsMap` which happens *after*
Gradle input snapshotting, so these arguments are not actually considered
for calculations of the task input hashes.

To illustrate this point better, let's way we have
```kotlin
@CacheableTask
open class MyTask: DefaultTask() {
    @get:Input
    val myInput: MutableList<String> = mutableListOf();

    @get:OutputFile
    val myOutput: File = File(project.buildDir, "foo.txt")

    @TaskAction
    fun doSomething() {
        myOutput.writeText(myInput.toString())
    }
}

tasks.register("myTask", MyTask::class.java) {
    myInput.add("abc")
    doFirst {
        myInput.add("123")
    }
}
```

when this runs, it creates `foo.txt` with contents of `[abc, 123]`.

If I run the build with `org.gradle.caching.debug=true`, it logs
```text
Appending input value fingerprint for 'myInput' to build cache key:
36f1f7201d2456f0db9c8b2028931377
```

Now, I modify `doFirst {` to add `"1234"` instead, and run it again. As expected,
I get `[abc, 1234]` inside `foo.txt`, however the caching log prints
```text
Appending input value fingerprint for 'myInput' to build cache key:
36f1f7201d2456f0db9c8b2028931377
```

(note, this is the same value as above)

This essentially means that modifications to Gradle inputs inside `doFirst`
*do not* affect task snapshotting, but they do affect task execution. Hopefully,
this gives you an idea of why this is a big foot-gun.

The reason why this works for AGP is that they do additional work
to add Compose Compiler Plugin `FileCollection` as an input to `KotlinCompile`
tasks
```kotlin
task.inputs.files(compilerExtension)
    .withPropertyName("composeCompilerExtension")
    .withNormalizer(ClasspathNormalizer::class.java)
```
This makes changes to the compiler plugin invalidate the task if the classpath
actually changes. AGP uses [ClasspathNormalizer](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/ClasspathNormalizer.html)
to makes Gradle ignore the absolute paths that I originally saw in `doFirst` print-outs.