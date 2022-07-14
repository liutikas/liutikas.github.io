---
layout: post
title:  Stop Wasting Time - Adopt Gradle Remote Build Cache
header: header-eng
---

In [Caches Everywhere]({% post_url 2022-04-19-Caches-Everywhere %}) post I called
out Gradle Remote Build Cache (RBC) as one of the types of caches in Gradle.
If you are on a project that has more than a handful of people, and you are
*not* using RBC, then you are wasting precious engineering time. This post will
explain how to get it working using a [Google Cloud Storage bucket](https://cloud.google.com/storage/docs/key-terms#buckets).

RBC allows you for a given set of inputs to only run this task once across
*all* of your developers. After RBC is filled with the task result, the
rest of the developers directly download the output from RBC instead of
rerunning this task again. This means you get to skip heavy hitters like Kotlin
compile, JVM tests, and Android Lint. This results in huge time savings!

The AndroidX team has created [GCP Gradle Build Cache Plugin](https://github.com/androidx/gcp-gradle-build-cache)
to make it easy to get going. [README.md](https://github.com/androidx/gcp-gradle-build-cache#readme)
explains [how to apply the plugin](https://github.com/androidx/gcp-gradle-build-cache#using-the-plugin)
in your `settings.gradle(.kts)` and [how to set up a Google Cloud Storage bucket](https://github.com/androidx/gcp-gradle-build-cache#setting-up-google-cloud-platform-project).

The flow that we found works best for AndroidX project is to only push to RBC
from the post-submit CI, whereas pre-submit CI and local developers get read
access to RBC. This makes sure that the cache does not have a lot of churn
as PRs and local changes are being built.

The whole setup ends up being:
```Groovy
plugins {
  id("androidx.build.gradle.gcpbuildcache") version "XXX"
}
import androidx.build.gradle.gcpbuildcache.GcpBuildCache 
import androidx.build.gradle.gcpbuildcache.GcpBuildCacheServiceFactory
buildCache {
    registerBuildCacheService(GcpBuildCache, GcpBuildCacheServiceFactory)
    remote(GcpBuildCache) {
        projectId = "..."
        bucketName = "..."
        push = isPostSubmitCi
    }
}
```

For a large project with hundreds of modules like AndroidX, we went from taking
minutes to seconds for most tasks both locally and in CI. This lets our
engineers stay in the flow and be more productive. It terms of cost,
it is less than <$500 per month with hundreds of CI builds a day and a long
list of contributors.

Here is an example of how tasks get a lot of cache hits:

![Screenshot of build scan showing cache hits](/assets/2022-07-13-cache-hits.png)

You can also browse the [AndroidX Gradle Enterprise page](https://ge.androidx.dev)
to see the remote cache hits for yourself instead of just taking my word for it.
