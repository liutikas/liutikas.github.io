---
layout: post
title: Kotlin Double Bind
header: header-eng
---

It is a great idea to give your library consumers maximum possible flexibility. In my post about [conservative librarian]({% post_url 2025-01-10-Conservative-Librarian %})
I suggested to keep your Kotlin language level as low as possible, as that allows consumers to use any Kotlin compiler
version between the library language level and the latest. Ideally, you only bump your Kotlin language version when you
need a new language feature to make your library better, for example picking up 2.4 to support Kotlin context parameters.

At the same time, as a library author you should also use the latest versions of tools, including the latest Kotlin
Gradle Plugin (KGP) and the Kotlin compiler. Using latest versions gives you performance improvements, bug fixes, and most
importantly helps you stay up to date as many small upgrades are easier than one giant leap upgrade. If you want more
evidence, I presented [a full talk on battling Kotlin upgrades at KotlinConf 2026](https://youtu.be/bavxLLEGX4w?si=8ds66VhAugUEEDE5).

Kotlin team has a policy where [KGP supports producing binaries that go back as far as 3 minor versions](https://kotlinlang.org/docs/kotlin-evolution-principles.html#evolving-the-binary-format).
That decision is really helpful for allowing library authors to do the best for their users. Sadly, I believe this range
is no longer sufficient. As the language matured and gained adoption, the spread of KGP versions used by the end
applications is very broad. For example, I know of folks happily using Kotlin 1.9 without any urge to upgrade as that
version has everything they need. I have also seen library authors convert their code from Kotlin to Java to avoid
imposing Kotlin versions due to this limited range.

To draw a comparison, `javac` 26 allows you to create a jar that targets Java 8. This policy allows library authors to
use the latest tools and still be generous to their consumers that have their own reasons to stick to old versions of JDK.
Today, when using KGP `2.4.0` you can only produce binaries with Kotlin language `2.1`, which forces library consumers
to use KGP `2.1.0` released on November 2024 or newer.

Library authors also often build tools to go with the libraries, for example `androidx.room` ships a library runtime
and a Kotlin Symbol Processing (KSP) code generator, `androidx.compose.ui` libraries ship along with inspection artifacts
used by Android Studio to debug the UIs. Tool authors face the opposite presure than library authors to support the
*latest* version of Kotlin language often forced by the need to use `kotlin-metadata-jvm`. `kotlin-metadata-jvm` supports
[safely reading one version forward](https://kotlinlang.org/docs/metadata-jvm.html#read-and-parse-metadata), so if you
use `2.4.0` you can read Kotlin binaries with `2.5` language level. This need *forces* tool authors to be on the latest
version of KGP to give the tool consumers the maximum possible flexibility.

These two forces combined turned into a Kotlin double bind, supporting old Kotlin language version pushed you to use
an old KGP version, and features/bugfixes/tool needs pushes you to use the latest KGP version. Either choice hurts one
side of consumer vs publisher.

I think it is time for Kotlin to follow suit and expand their support window for producing binaries targeting older
language versions. This would drastically help library and tool authors and thus the entire ecosystem to continue to
flourish as Kotlin adoption grows.

Thoughts?
