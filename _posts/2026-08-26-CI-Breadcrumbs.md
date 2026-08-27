---
layout: post
title: CI Breadcrumbs
header: header-eng
---

Hitting a failure in continuous integration (CI) can be really frustrating, especially if that failure happens in a flaky
way. One way to resolve it is to try the same command locally in as close enough of a state as possible and repeat that
until you hit the same issue. Not only that is time-consuming, but it might also not happen at all due to minor
differences between your local machine and CI. Luckily, there is a way to leave yourself breakbumbs to help identify
what happened in the CI run without a local reproduction.

It is a great idea to have a build timeout, however when you hit this timeout, it can be very unclear on why that
happened, especially if the logging is sparse at the time the build hangs. The insight here is that you can [capture
`jstack` dumps](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:busytown/impl/showJavaStacks.sh)
for all Java processes running in your CI right before you terminate the build process. You can then include the
`jstack` in the failure log to ease debugging.

Similarly, you can add `-XX:+HeapDumpOnOutOfMemoryError` to your `org.gradle.jvmargs` to make sure Java dumps heap state
for Gradle going out of memory. Then you can store that heap dump to help debug your build and test processes.
 
If you combine these breadcrumbs with a system that allows your to rapidly search all build logs for the
failed builds (discussed in my ["Moments When Things Go Wrong - KotlinConf 2026"](https://youtu.be/bavxLLEGX4w?si=8ds66VhAugUEEDE5)
talk) then you can build a pattern of the timeouts and out of memory errors. It can also give you a way to see when it
started happening.

These were exactly the tools that helped us find a deadlock in `androidx.tracing` library. The issue was caused by
[a performance optimization](https://r.android.com/4236477). We saw timeouts in tracing JVM tests, used failed build
search to find when it started, and finally used `jstack` traces from various builds to identify the source of the deadlock.