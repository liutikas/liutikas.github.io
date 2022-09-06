---
layout: post
title:  What's inside that Gradle zip?
header: header-eng
---

Gradle makes it quite easy to start a new project with a [Gradle wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html)
and the `gradle-wrapper.properties` file that points to a Gradle distribution
zip:
```
distributionUrl=https\://services.gradle.org/distributions/gradle-7.5.1-bin.zip
```

Most people ever interact with this part of their build when they bump a Gradle
version, but have you ever peeked inside this zip to see what you are downloading?

![Screenshot of Gradle zip contents](/assets/2022-09-06-zip-contents.png){:height="200px"}

In [Gradle Security Considerations]({% post_url 2022-05-13-Gradle-Security-Considerations %})
I called out that you likely want to set `distributionSha256Sum` to make sure
you are getting the same zip every time. [Gradle is an open-source project](https://github.com/gradle/gradle),
so we can go one step further - we can validate that the zip is reproducible.
To do that, we follow the following steps:
1. Clone [Gradle git repository](https://github.com/gradle/gradle)
2. Check out the release tag for the version we care about (in this case [7.5.1](https://github.com/gradle/gradle/releases/tag/v7.5.1)).
3. Set your `JAVA_HOME` to a distribution of JDK 11 (in my case `Zulu 11.0.13`)
4. `./gradlew :distributions-full:binDistributionZip -PfinalRelease`
5. Compare [Gradle published zip](https://services.gradle.org/distributions/gradle-7.5.1-bin.zip) and `subprojects/distributions-full/build/distributions/gradle-7.5.1-bin.zip`

When doing the comparison I found that everything inside was identical except for
2 jars: `lib/fastutil-8.5.2.min.jar` and `lib/gradle-base-services-7.5.1.jar`.
`fastutil` jar difference is [due to keeping the timestamps in the `Minify` transform](https://github.com/gradle/gradle/issues/21848).
`gradle-base-services` difference is due to `org/gradle/build-receipt.properties`
file including a build timestamp. If both of these issues get fixed, future
versions of Gradle should result into identical zips when comparing Gradle published binary
and a local build of the release commit.

Note, this distribution zip contains more than just 94 jars that are built from
sources in the Gradle GitHub repository. This zip also has 139 jars that Gradle build
grabs off of maven repositories and the reproducibility of these is more tricky
to verify.
