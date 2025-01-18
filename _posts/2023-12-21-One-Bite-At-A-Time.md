---
layout: post
title:  One Bite At A Time
header: header-eng
mastodon: 111621422904232267
---

As part of our regular Gradle build scan investigation we looked at an AndroidX
build that should have been largely a 100% remote cache hit. We found that we
were in fact hitting the cache as expected (yay!), but the build timeline had
a long task that hit the cache and yet was taking way longer than any other task
(nay!).

![Screenshot of Gradle timeline showing one task taking a long time](/assets/2023-12-21-timeline.png){:width="600px"}
([timeline](https://ge.androidx.dev/s/4zzw27ril4hw6/timeline))

Taking a deeper look, we noticed that this task has a fairly large 60.7MB output,
but it is downloading it at a measly 750KBps.

![Screenshot of Gradle task detail view](/assets/2023-12-21-task-details.png){:width="600px"}
([task details](https://ge.androidx.dev/s/4zzw27ril4hw6/timeline?details=tbyyxnqlyzy3c&expanded=WyI3IiwiMSJd&page=126))

Luckily it was easily reproducible locally. Trying to download the exact same
cache entry using `gsutil` directly took <2 seconds, so we clearly had an issue.
Taking a cursory look at a network activity on the machine showed that even
Gradle build cache download takes little time, but the cache entry is not
returned for over a minute after.

![Screenshot of network manager showing one big burst](/assets/2023-12-21-network.png){:width="600px"}

Then following profiling that I've blogged about before, it was clear that we
were spending all of our time in `FileInputStream.read`.

![Screenshot of a flamegraph for fetching the cache entyr](/assets/2023-12-21-flamegraph.png){:width="600px"}

And then it dawned on us, we were not doing any sort of buffering for this!!!
[A small PR later](https://github.com/androidx/gcp-gradle-build-cache/pull/40)
we went from [82s (750KBps)](https://ge.androidx.dev/s/q36vklz4xyytw/performance/build-cache?anchor=eyJpZCI6InJlbW90ZS1oaXQifQ&cacheDetails=remote-hit)
to [1.7s (36.8MBps)](https://ge.androidx.dev/s/qn2ueapsujhjq/performance/build-cache?anchor=eyJpZCI6InJlbW90ZS1oaXQifQ&cacheDetails=remote-hit)
to download the same remote cache entry.

Moral of the story? Always be buffering your input streams! How did I not learn
this from [I/O profiling]({% post_url 2023-03-02-Profiling-Your-IO %}) earlier
this year?
