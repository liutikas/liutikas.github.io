---
layout: post
title:  Cache, Cache, No Cache - When to use Gradle Build Cache?
header: header-eng
mastodon: 115451769649806985
---

In the light of [CVE-2025-36852](https://nvd.nist.gov/vuln/detail/CVE-2025-36852) I figured it would be a good time to
discuss the use of Gradle Build Cache in various set-ups.

In androidx we have the following policies regarding the build cache:

**Developers have read-only access to the remote cache**

We want developers to gain build performance so we allow them to use remote cache for their local development. They are
also able to write to the local build cache to help them out in case they switch branches. It might be tempting to allow
writing to remote cache as it could speed up your CI runs, but there are 2 reasons we don't allow it: 1. a developer
could intentionally or unintentionally (e.g. malware) [poison the cache](https://en.wikipedia.org/wiki/Cache_poisoning)
by pushing malicious task outputs 2. pushing to remote cache comes with resource overhead, and we prefer to not use these
on developer machines.

**Pre-merge CI has read-only access to the remote cache**

Similar to the local developers we want performance gains, and we also want to avoid poisoning the cache. Pre-merge CI
has as little trust as local developer machine as the code here potentially has had no review yet.

**Post-merge CI has read and write access to the remote cache**

This is the only part of the build that we trust to push to the remote cache. The reason we have high confidence in these
builds is that the code has been reviewed by at least 1 more person and the chances for malicious cache entries decreases.

**Release CI has no cache access**

The release artifacts is our product. These artifacts are how AndroidX is consumed by all of our users. Given that, it is
extremely important to prevent malicious attacks as much as possible. To help that, our release CI builds everything
from scratch without any remote cache. The time saved is not worth the cost of potential failure.

AndroidX runs release CI for every commit, but you may choose a less frequent schedule.

**Additional considerations**

You can consider a *separate* remote build cache for pre-merge CI that is only used for pre-merge and does get reads and
writes. We chose not to go with this as the gains are minimal for our workflow patterns.
