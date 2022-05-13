---
layout: post
title: Gradle Security Considerations
header: header-eng
---

Running any binary on your computer is inherently risky and `gradlew` is no exception, especially since it directly affects products that you ship to your customers. I wanted to share a couple of tips to improve the sitation.

- Do not hastily run `./gradlew` on an external project you checked out.
  - Replace their Gradle wrapper with a one you trust.
  - Squint at `buildSrc/` and `*.gradle` files before running.
  - Run in a VM if possible
- Validate [Gradle wrapper JAR](https://github.com/marketplace/actions/gradle-wrapper-validation) checksum every time you upgrade and [using GitHub Action if you use GitHub](https://github.com/marketplace/actions/gradle-wrapper-validation).
- [Set `distributionSha256Sum=XXX` in `gradle-wrapper.properties`](https://docs.gradle.org/current/userguide/gradle_wrapper.html#configuring_checksum_verification) to validate that `gradle-X.X-bin.zip` actually matches [SHA256 provided by Gradle](https://gradle.org/release-checksums/).
- Use a trustworthy and up-to-date JDK to run Gradle (e.g. [Amazon Corretto](https://aws.amazon.com/corretto/) or [Zulu](https://www.azul.com/downloads/?package=jdk))
- Carefully pick your Gradle plugins, annotation processors, Kotlin and Java compiler plugins. Each one brings in risk, especially if it is a closed source artifact.
- Enable [Gradle dependency signature verification](https://docs.gradle.org/current/userguide/dependency_verification.html#sec:signature-verification) to make sure that artifacts you use (especially on build classpath) are signed with a trusted key ([example configuration file from AndroidX](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:gradle/verification-metadata.xml)). It will help catch man-in-the-middle attacks and any incidental artifact modifications. It is a fairly painful feature to set up ([a couple of Gradle bugs](https://github.com/gradle/gradle/issues?q=is%3Aissue+is%3Aopen+signature+verification)), but the gains are high.
  - Complain to projects that do not sign or only partially sign their artifacts (throwing a rock into my own AndroidX garden), for example [Kotlin Gradle Plugin](https://youtrack.jetbrains.com/issue/KT-50957/Enabling-Gradle-signature-verification-on-a-basic-Kotlin-project) and [Android Gradle Plugin](https://issuetracker.google.com/215430394)
  - Sign your libraries, if you ship any.
- Never publish locally built artifacts, ideally it all comes from a reproducible clean Virtual Machine build.

This will not solve all of your problems, but it will get you in a better shape.
