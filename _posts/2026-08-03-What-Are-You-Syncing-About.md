---
layout: post
title: What Are You Syncing About?
header: header-eng
mastodon: 117033832563717682
---

![Sync Project with Gradle Files button in Android Studio](/assets/2026-08-03-elephant-button.png)

Did you ever wonder what happens when you press that elephant button ("Sync Project with Gradle Files") in Android
Studio?

Before we dive in on how it works, let's consider what an IDE needs from Gradle to work well. Here is a small set of
details it retrieves about the project:

- What is the list of all the subprojects?
- What plugins are applied to each project?
- Where are all the source directories?
- What are the dependencies for each compilation unit?
- What are the target Java and Kotlin versions?
- What flags are passed to Kotlin compiler (e.g. `allWarningsAsErrors = true`)

and many more. These details can change with `build.gradle.kts`, `settings.gradle.kts`, `gradle.properties`, `buildSrc`,
and many other changes, so any time these get invalidate you will get a dreaded:

![Gradle files have changes since last project sync. Sync now](/assets/2026-08-03-sync-now.png){:width="800px"}

For a long time I simply assumed it was doing the same thing as a commandline `gradlew` invocation invoking some tasks
and turns out that this is only partially true. In this post I wanted to share some details on what really happens there.

The first difference is in how Gradle gets invoked. Android Studio does not invoke Gradle through commandline, instead
it uses Gradle's [Tooling APIs](https://docs.gradle.org/current/userguide/tooling_api.html) to orchestrate Gradle and
get back the information it needs for IDE to function.

Then we have two familiar Gradle [build lifecycle phases](https://docs.gradle.org/current/userguide/build_lifecycle.html):
initialization and configuration. This is effectively the same work you would see in commandline invocations, and it is
currently (without [Gradle Isolated Projects](https://docs.gradle.org/current/userguide/isolated_projects.html) enabled)
entirely serial, namely the larger your projects the proportionally slower the configuration.

The next phase in commandline invocations is the task execution. That phase is generally not used for IDE Gradle sync.
A big exception is Kotlin Multiplatform (KMP) that takes a shortcut and utilizes tasks to generate cinterop code to
allow IDE to resolve these symbols. Sadly, this is blocking and not build cacheable, so [hopefully it eventually gets fixed](https://youtrack.jetbrains.com/issue/KT-87309/).

In IDE Gradle sync we then run model building phase. This process is complex, but in a layman's terms, each ecosystem
gets a chance to prod Gradle APIs to get the parts relevant to that ecosystem. In the AndroidX project we build JVM,
Kotlin, KMP, and Android models. Sadly, every model builder except for Android is done in a serial way. That means that
even with Gradle Isolated Projects enabled, you will then be bottlenecked by the model builders. You can chime in on
[this YouTrack bug](https://youtrack.jetbrains.com/issue/KTIJ-28043) if you'd like this to improve.

You can use [Gradle's internal operation tracing]({% post_url 2026-02-25-org.gradle.internal.operations.trace %}) to
get a sense where your IDE sync time is being spent. Here is an example for AndroidX:

![Perfetto trace of AndroidX IDE sync](/assets/2026-08-03-trace.png){:width="800px"}

You can take a look at [a full perfetto trace](https://ui.perfetto.dev/#!/?s=193d126c3d767f4bd05fdb08e995f8f2a8102edf)
for an AndroidX IDE sync (note this will be slow as the trace is >2GB).

So to recap: call through tooling APIs > initialization > configuration > tasks (likely skipped) > model building. At
that point Gradle is no longer used, but IDE might still do work such as indexing your source code and dependencies.

I hope this sheds some light into this topic!
