---
layout: post
title:  Pom Pom Pom 🎶 - Have we reached the end of life?
header: header-eng
mastodon: 114673454301919245
---

Note: This post has been edited on July 2, 2025 to clarify details about POMs.

## Brief history of POM file proliferation

Maven build system uses [`.pom`](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html) (POM) files
to define how a library should be built and what other libraries this project depends on. The same POM file is reused
when this library is published to a remote Maven artifact repository (e.g. MavenCentral), so the consumers can get these
artifacts and all dependencies needed to use it. It is a neat idea allowing consumers to skip building every single
dependency from scratch, which is the way that is common in other ecosystems.

The Maven build system itself is a generic automation tool, but by default it focuses of Java/JVM needs. Therefore,
the POM files also take a largely Java/JVM-centric view, for example, if not otherwise specified, the default
artifact's and dependencies' format is a `.jar` (JAR).

Remote Maven artifact repository and Maven POM file popularity has lead to other build systems, such as Gradle, to take
advantage of this system and build adapter layers for the publishing and the consumption of already published Java
libraries. That means that Gradle is able to parse and interpret POM files on Maven artifact repository
(such as MavenCentral) and use them regardless of the build system used to build those artifacts. Similarly, when
publishing Java libraries, Gradle is able to generate POM files from Gradle's own `build.gradle.kts` build definition
files allowing other users to consume these libraries. POM files that were originally a Maven build system specific
format turned into a very valuable compatibility layer / de facto standard for JVM build systems to inter-op with each
other, thus allowing JVM ecosystem to flourish.

Sadly, POM format is not a perfect solution for remote artifact consumptions, and this post will share a few reasons
why it might be the case.

## Assumption about a single artifact

POM files can only describe a single artifact for a given Maven coordinate (for example `com.example:example:1.0.0`)
and by default that artifact is a JAR (`example-1.0.0.jar`), unless specified otherwise via entry like
`<packaging>aar</packaging>`. This is good enough for a large number of JVM use-cases, but there are some use-cases
that this does not work with:
- debug vs release versions
- compile vs runtime versions
- per-CPU architecture versions
- JVM target versions (Java 8 vs Java 21)
- JVM vs Android compatible

Given you can only have a single artifact, library publishers are forced to pick a default version, which does not
always work, e.g. it might be hard to pick an artifact that works for every OS.

One workaround for this is using [classifiers](https://maven.apache.org/pom.html#:~:text=The%20classifier%20distinguishes%20artifacts%20that%20were%20built%20from%20the%20same%20POM%20but%20differ%20in%20content)
that allows you to specify `com.example:example:1.0.0:android`. This notation allows consumers to resolve `example-1.0.0-android.jar`
instead the default `example-1.0.0.jar`. This works until you want to publish another library
`com.other:other` that depends on this existing library. There is only a single `<dependencies>` block in the POM
file, so when `com.other:other` adds an entry for `com.example:example` it has to pick specify
`<classifier>android</classifier>` or it will default to the main artifact. That means someone using `com.other:other`
can either have it working for JVM or Android, but not both.

Another approach might be to encode information in the artifact version. For example, you could have
`com.example:example:1.0.0-jvm` and `com.example:example:1.0.0-android`. User can choose the right one and have it work.
This even is able to avoid the issue with the classier approach as `com.other:other:1.0.0-jvm` can depend on
`com.example:example:1.0.0-jvm` and `com.other:other:1.0.0-android` can depend on `com.example:example:1.0.0-android`.
Sadly, this hits a big stumbling block when you have multiple versions being pulled in by different dependencies. By
default, Maven will result to the highest version between the dependencies, e.g. if you have `1.0.0-android` and
`1.0.0-jvm` it will actually pick `1.0.0-jvm` as `-j` is alphabetically newer than `-a`. This requires consumers to do
careful version management to get the correct versions.

Yet another way could be to split your artifacts to use multiple Maven coordinates `com.example:example-jvm` and
`com.example:example-android`. Similar to the approach that uses versions, this requires every library that builds on
top of this library to publish using multiple Maven coordinates as well: `com.other:other-jvm:1.0.0` and
`com.other:other-android:1.0.0`, but unlike that approach, it is a lot more obvious when the wrong artifact is selected
as there is no automatic version resolution involved, as you can see if `-jvm` or `-android` artifacts are used.

All the approaches above require flattening the resolution of the dependencies, namely every library dependency that is
added requires to match the splits of every other dependency. This results in a combinatorial explosion of all the
options when you consider all the various use-cases listed above (e.g. CPU architecture and JVM vs Android).

To make matters worse, it also means whenever a low level library needs to introduce multiple artifacts, every library
that builds on top of it needs to re-release to support it. This gets very complex as the depth of dependencies increases.

## A Better Way?

Gradle recognized these POM file limitations and chose to introduce [Gradle Module Metadata](https://docs.gradle.org/current/userguide/publishing_gradle_module_metadata.html)
`.module` (MODULE) files to support richer artifact publishing and consumption. It introduces a concept of attributes
that allows publishers to describe multiple variants of the artifact, for example `"org.gradle.jvm.environment": "android"`
vs `"org.gradle.jvm.environment": "standard-jvm"` and then the consumers provide these attributes to resolve the correct
artifact, while user only needs to specify `com.example:example:1.0.0` and it just works. Gradle puts these MODULE files
alongside the POM files to support non-Gradle consumers, but for cases where the default is hard to pick, we get back to
the issue described earlier.

Android Gradle Plugin (AGP) and Kotlin Gradle Plugin (KGP) (especially for Kotlin multiplatform) uses MODULE files to express the
variants of the artifacts produced using these plugins and users that use Gradle have it just work. *Note*, I do want to
acknowledge that when resolution fails, the debugging of attribute selection is a miserable experience.

## What about non-Gradle build systems?

This gets us to the crux of the problem. POM parsing and resolution is relatively simple to implement and thus many
build systems (e.g. Bazel) have built adapters to fetch artifacts for a Maven coordinate. Sadly, MODULE file parsing and
resolution requires either executing Gradle or painstakingly re-implementing and keeping up-to-date with [a specification](https://github.com/gradle/gradle/blob/master/platforms/documentation/docs/src/docs/design/gradle-module-metadata-latest-specification.md).
The resolution logic is scatted between Gradle core and third-party plugins such as AGP and KGP, where these plugins
keep changing their attribute set and [compatibility rules](https://docs.gradle.org/current/userguide/variant_aware_resolution.html#step_1_find_compatible_candidates).

Bazel, for example, seems to be headed in a direction of using [Gradle in an isolated process](https://github.com/bazel-contrib/rules_jvm_external/pull/1357),
but it is not clear if this will merge and what the performance characteristics of this system will be.

In the meantime, POM user experience keeps getting worse due to the issues describe in this post, as well as [a variety
of bugs in Gradle and KGP](https://github.com/bazel-contrib/rules_jvm_external/issues/1376#issuecomment-2968205021).
Have we reached the end of life for POMs?

## Addendum

Maven is a great build tool and has had a huge impact on the Java/JVM ecosystem. This post is by no means a suggestion
to avoid using it. The goal is more to discuss whether we could have more flexibility using something else to support
an idea of artifact variants/flavors.
