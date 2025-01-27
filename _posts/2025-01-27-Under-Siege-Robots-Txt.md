---
layout: post
title:  Under Siege? robots.txt!
header: header-eng
---

We maintain a handy [androidx.dev](https://www.androidx.dev) artifactory service that allows developers to try out
androidx libraries at `HEAD` to validate fixes or just live on a bleeding edge. In fact, it is also used by
Android Studio team to [serve Android Gradle Plugin](https://androidx.dev/studio/builds) at `HEAD` too.

Last week, I looked at our traffic to this service, and I was kind of blown away that we were spiking to over 600
requests per second. The request pattern was cyclical with about 15-minute period. A deeper analysis showed that
majority of this traffic was coming from bots (e.g. `meta-externalagent`, `Barkrowler`, and others) going through our
endless build pages that we create every time a change lands and navigating to every `.xml`, `.pom` and other files.
All of was creating a heavy load on our service while providing no benefit as all of these builds are intended to
be short-lived.

We added a [`robots.txt`](https://www.androidx.dev/robots.txt) to tell bots to stop crawling our pages and within a day
our request traffic dropped drastically.

![A graph showing drastic drop in request traffic](/assets/2025-01-27-traffic.png){:width="800px"}

It seems that we are largely back to real users and some bots that don't respect `robots.txt`.
