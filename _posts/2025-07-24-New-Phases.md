---
layout: post
title:  New Phases - Gradle Configuration Store and Load
header: header-eng
mastodon: 114910285932492916
---

For a long time Gradle had [three distinct build phases](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases)
which in simplified terms were:

![The legacy phases of the build](/assets/2025-07-24-legacy-phases.svg)

1. **Initialization**: build a graph of projects
2. **Configuration**: build a graph of tasks
3. **Execution**: run the requested tasks and their dependencies

These phases are run serially, and historically they were never skipped.

As everyone's builds grew in size and complexity, the configuration phase was becoming a sizeable chunk of the overall
build time. To make matters worse, most of the time developers had no changes to the projects meaning that the results
of initialization and configuration were identical to previous Gradle runs, but Gradle would repeat the work anyway.

This observation turned into a multi-year quest for Gradle and the plugin ecosystem to support [configuration cache](https://docs.gradle.org/current/userguide/configuration_cache.html)
feature that allows Gradle to skip the configuration phase if it knows the task graph is unchanged saving all of this
wasted time. With Gradle `9.0.0` [configuration cache is becoming a preferred execution mode](https://gradle.org/whats-new/gradle-9/#:~:text=a%20different%20lifecycle.-,Configuration%20Cache,-The%20Configuration%20Cache)
which is Gradle's way of saying they cannot make it the default as the long tail of plugins still don't support it (YO fix your plugins!), but
it will become the default in `10.0.0`.

To make configuration cache work Gradle sneakily added two new phases to the build without Gradle docs talking about -
Configuration Cache storing and loading phase (let's call it `CC store` and `CC load`).

![The new phases of the build](/assets/2025-07-24-new-phases.svg)

In a configuration cache hit case things are pretty awesome as you just load the configuration cache and start running the
tasks. [Starting Gradle 8.11 CC load got even faster as it is now loaded in parallel](https://docs.gradle.org/8.11/release-notes.html#:~:text=Faster%20Configuration%20Cache%20with%20parallel%20load%20and%20store).
Most of the time this is well worth it as configuration phase can take several minutes for a large project and CC load
is 3-5 seconds. For example, androidx has 1183 projects and a large task graph configuration takes about 4 minutes, but
CC load is only about 4 seconds.

In a configuration cache miss case you get both CC store and CC load that delay the start of task execution. As of
Gradle 8.14.3 by default the CC store is a largely single threaded operation and for androidx it takes about 70 seconds.
That means in CC miss case we add about 74 seconds to already 4+ minutes configuration making this very costly.

Lucky, starting Gradle 8.11 you can now set [`org.gradle.configuration-cache.parallel=true` property](https://docs.gradle.org/8.11/userguide/configuration_cache.html#config_cache:usage:parallel)
that enables parallel CC store. In androidx build it allowed our CC store to go from 70 seconds to 11 seconds effectively
shaving off almost a minute from our ephemeral CI where CC was not hit ever, putting our CC overhead to about 15 seconds.

![Build scan showing serial CC](/assets/2025-07-24-serial-cc.png){:width="800px"}

![Build scan showing parallel CC](/assets/2025-07-24-parallel-cc.png){:width="800px"}

![A graph showing a drop in non-execution time](/assets/2025-07-24-graph.png){:width="800px"}

Note, parallel CC store is risky because if any plugin you use has assumptions about execution order or single-threadedness
you might get `ConcurrentModificationException` or other similar issues, so your mileage might vary.

