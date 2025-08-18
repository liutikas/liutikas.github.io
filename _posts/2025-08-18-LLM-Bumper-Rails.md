---
layout: post
title:  LLM Bumper Rails - New Problems Same Techniques
header: header-eng
---

High quality products tend to make users happier, but making high quality software is not an easy task. Static analysis tools like 
Android Lint and Error-prone have been invaluable in making sure mistakes have a harder time to creep into your codebase.
AndroidX, Android platform, and Android ecosystem at large have embraced these tools and created a ton of great custom
checks. Compose libraries, for example, ship with a variety of bundled Android Lint checks helping users to avoid
pitfalls in trickier scenarios, allowing you to spend more time on making stunning UIs instead of debugging the code.

In addition to static analysis it is also helpful to add other build time checks. For example, you can catch issues earlier
by enabling `warningAsErrors` for Kotlin or Java compilations or enabling [`enableStricterValidation`](https://docs.gradle.org/nightly/javadoc/org/gradle/plugin/devel/tasks/ValidatePlugins.html#getEnableStricterValidation())
in `java-gradle-plugin`. In the AndroidX project we built even more checks on top of it, for example we
[validate that there are no overlapping outputs](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:buildSrc/private/src/main/kotlin/androidx/build/ListTaskOutputsTask.kt;l=32;drc=c6155c0227c25d3cbdbeafb3e42418b5d843c5df)
across our tasks to ensure that we don't get accidental invalidations of tasks. Similarly, we make sure that our
[build runs no unexpected tasks when it is run twice in a row](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:buildSrc/private/src/main/kotlin/androidx/build/uptodatedness/TaskUpToDateValidator.kt;l=182;drc=c6155c0227c25d3cbdbeafb3e42418b5d843c5df).
On top of that, we have checks that validates that generated artifacts contain all expected contents, for example `.pom`
files for maven artifacts.

Outside the automated checks it is handy to have manual verification. In AndroidX project we have [a script](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:development/validateRefactor.sh;drc=b4c76e49a441c2f21a815ac4a435923eac089c02)
that runs the build with and without a specified PR to ensure that build outputs have no unexpected changes for PRs that
refactor build logic. All these guardrails allow us to make changes to our build logic with high confidence that things
will not break.

With the rise of LLM generated PRs these checks are becoming even more important. LLMs learned to generate code using the
Internet as the source, so it picked up all the "creative" ways of solving problems too including all of the bugs and
novice mistakes. There has been a big shift for humans to spend more time reviewing code and as that volume increases
there is a fatigue that can let through buggy code. All the tools and checks listed above work just as well on all PRs
despite what created it and thus comes extremely handy to help you.

My suggestion is to double down on static analysis and build checks. Adding a custom check could save yourself hours of
work in the long-term. My general rule is if I catch an issue 3 times in code review in the last 3 months, then that is
the time to spend time automating the way to catch it. In fact, that is the origin story of [`androidx.lint:lint-gradle`](https://developer.android.com/jetpack/androidx/releases/lint).
I hope you are already using it on your binary Gradle plugins.
