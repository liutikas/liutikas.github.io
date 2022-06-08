---
layout: post
title: Containing Naughty Gradle
header: header-eng
---

Gradle, just like any other Java application, can open up arbitrary network connections
and initiate downloads and uploads. That is scary as it creates risks for your
local/CI machines and could also affect artifacts you ship to your customers.
In my post on [Gradle Security Considerations]({% post_url 2022-05-13-Gradle-Security-Considerations %})
several of the steps help by giving you higher confidence that Gradle executable
and all the dependencies have not been modified. However, that doesn't prevent
Gradle plugins and invoked tools from intentionally or unintentionally
fetching from the network.

One way to stop Gradle from using the network is to use
`--offline` flag. This requires your entire build to be completely local
including having all of your maven dependencies pre-downloaded (e.g. in git).
This setup is quite painful to manage and thus doesn't work for everyone.
Additionally, `--offline` is merely a hint therefore plugins can still ignore it.
For example, Android Lint tasks were [accidentally downloading files when running one
of the checks](https://issuetracker.google.com/225244932).
Similarly, Robolectric fetches Android system images on the first run
(but you [can disable it](http://robolectric.org/configuring/#:~:text=SDKs%20are%20enabled.-,robolectric.offline,-%E2%80%94%20Set%20to%20true)
if you point it to your own local repository).

Luckily, there is a better solution for containing Gradle -
[nsjail](https://github.com/google/nsjail).
It is a general tool that can sandbox processes on Linux. nsjail can be used
to limit Gradle to only be able to access the network in a way that you expect.
You can list all of your maven repositories, remote build cache server and any
other resources to enforce that your build can pass without any other access.
[AndroidX has been using nsjail](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:development/sandbox/run-without-network.sh)
as part of the Kotlin Native builds exploration to ensure that it doesn't
download arbitrary executables without going through Gradle ([it currently
does](https://youtrack.jetbrains.com/issue/KT-52567/Use-Gradle-dependency-management-for-downloading-KotlinNative-compiler-when-compiling-with-Gradle)).
There are more options listed at [nsjail.dev](https://nsjail.dev/).
