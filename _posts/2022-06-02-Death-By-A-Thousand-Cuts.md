---
layout: post
title: Gradle Configuration - Death by a Thousand Cuts
header: header-eng
---

In [Profiling - The Good Kind]({% post_url 2022-06-01-Profiling-The-Good-Kind %})
post I noted that Gradle [configuration phase](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases)
largely runs on a single thread with each project being configured (plugins applied,
build scripts executed) serially. This means a single 1ms bit of logic turns
into 500ms of configuration time if you have 500 projects in your `settings.gradle`.

One of the easy ways to improve here is to apply plugins only when they are
needed, namely avoiding `allprojects { apply from: foo.gradle }`
or applying plugins just in case you'll need it at some point.

Another consideration is to make your [Gradle configuration as lazy as possible](https://docs.gradle.org/current/userguide/lazy_configuration.html)
moving to properties and providers so that the heavy work is only done when
your task is executed. You should avoid I/O (e.g. reading contents of a file),
calling external processes (e.g. getting git commit), making network calls, or
other heavy computations during the configuration phase. Don't hesitate to file
bugs to plugins that haven't migrated yet - it is time for them to move to modern
Gradle APIs.

Next, for the work that *must* happen in configuration evaluate whether it
would be enough if was done on the rootProject only. If that is not the case,
then see if you can use a [build service](https://docs.gradle.org/current/userguide/build_services.html)
to share part of the work between all the tasks. In AndroidX we use a build
service to load a TOML file that holds some build configuration which gets
computed with the first call to the service and the rest of the tasks get the
cached value.

Finally, Gradle configuration phase is a death by a thousand cuts, so you
*have* to be as efficient as possible, especially if you ship a public Gradle
plugin. For example, a call to `String.format` from a [`Precondition`
statement in a hot-path in Gradle caused wasted 168ms](https://github.com/gradle/gradle/issues/20883)
 and [another 24ms from the Android Gradle Plugin
code](https://cs.android.com/android-studio/platform/tools/base/+/8373d95e531aa6a8e8614fecdf9aab2e587465bb)
for the AndroidX configuration. A call to `Project.getProperties().get("FOO")`
instead `Project.findProperty("FOO")` was [costing us 704ms](https://android-review.googlesource.com/c/platform/frameworks/support/+/2108265).
Use of Kotlin reflection is also notoriously slow, Kotlin Gradle Plugin using
it to [calculate compiler arguments makes AndroidX configuration 1.3s slower](https://youtrack.jetbrains.com/issue/KT-52520/Remove-usage-of-reflection-from-CompilerArgumentsGradleInput).
This results in Gradle configuration time that is minutes long as thousands of
these mini slowdowns pile up.

I've created [a hotlist in the Google Issue tracker](https://issuetracker.google.com/hotlists/4259900),
in case you want to follow along on our ongoing work. It would be awesome to see
more developers tracing their configuration and filing bugs to the plugin
authors, so we can make the ecosystem better.

Side note, Gradle is working on making configuration phase run in parallel through
the [project isolation feature](https://gradle.github.io/configuration-cache/#project_isolation).
Sadly, we are looking at a fair amount of time before it can be adopted.
