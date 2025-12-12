---
layout: post
title:  Knocker-Uppers - PGP Validation of Gradle Wrapper and Distribution
header: header-eng
---

Many moons ago I wrote a post on [Gradle Security Considerations]({% post_url 2022-05-13-Gradle-Security-Considerations %})
that discussed various ways to protect yourself when using Gradle. We now have a new layer to add to those practices -
ability to validate signatures of Gradle wrapper jar and Gradle distribution zip.

First we've got to establish some background. In your Gradle build you have the following files:
```text
gradle/wrapper/gradle-wrapper.properties
gradle/wrapper/gradle-wrapper.jar
gradlew
gradlew.bat
```

`gradlew` is a script that uses `java` to run [`org.gradle.wrapper.GradleWrapperMain`](https://github.com/gradle/gradle/blob/master/platforms/core-runtime/wrapper-main/src/main/java/org/gradle/wrapper/GradleWrapperMain.java)
that is inside `gradle/wrapper/gradle-wrapper.jar`. This code reads `gradle-wrapper.properties` and then if needed
fetches the Gradle distribution based on `distributionUrl` and validates whether the `distributionSha256Sum` matches
the sha256 of the zip downloaded. This `gradle-X.X.X-bin.zip` contains all of Gradle code that gets extracted and then
the Gradle daemon process can be started that actually does that Gradle-ly things.

This might seem like a pretty secure system as an attacker that man-in-the-middles (MitM) the zip download would have to make
sure that the sha256 sum is identical which is a very hard task. Sadly, for any of this to be trusted you have to trust
that `GradleWrapperMain` does exactly what you would expect. A malicious actor might try to MitM `gradle-wrapper.jar`
download or craft a malicious PR (e.g. using a [homoglyph](https://en.wikipedia.org/wiki/Homoglyph) attack), at which
point `distributionUrl` could be changed and `distributionSha256Sum` validation could be skipped. At that point you have
no control of what happens in your build and what is inside the artifacts you create!

To help you Gradle offers a [`gradle/actions/wrapper-validation`](https://github.com/gradle/actions/tree/main/wrapper-validation)
GitHub Action, but that might not work for you depending on what CI you use.

The new development is that [now Gradle ships PGP armored (.asc) files for Gradle wrapper jars and distribution zips](https://github.com/gradle/gradle/issues/33940).
These are the same type of files that Gradle uses for validating your dependencies discussed in [Trust By Verify Dependencies]({% post_url 2024-12-12-Trust-But-Verify-Dependencies %})
post. You can find these `.asc` files in [Gradle distributions archive](https://services.gradle.org/distributions/).
This allows you to verify that `gradle/wrapper/gradle-wrapper.jar` and `gradle-X.X.X-bin.zip` were built by someone
who has Gradle's private PGP keys.

The process is quite simple, get the key [from Gradle.org](https://gradle.org/keys/), download the respective `.asc`
files, then you can validate them using:

```bash
gpg --import gradle_pubkey.asc
gpg --verify gradle-9.2.1-wrapper.jar.asc gradle/wrapper/gradle-wrapper.jar
gpg --verify gradle-9.2.1-bin.zip.asc  gradle-9.2.1-bin.zip
```

[In androidx we validate these signatures in CI](http://r.android.com/3885142) before we ever invoke any Gradle commands.

Happy defending!
