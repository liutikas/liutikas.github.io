---
layout: post
title:  Where is Your Signature?
header: header-eng
---

AndroidX project uses [Gradle signature verification](https://docs.gradle.org/current/userguide/dependency_verification.html#sec:signature-verification)
which is one of many [security practices you should follow]({% post_url 2022-05-13-Gradle-Security-Considerations %})
for your project as well.

When upgrading to `com.google.guava:guava` version `32.1.3` we spotted an odd signature verification behavior - we saw
that Gradle was not validating signatures for some Guava jar files, it was just validating checksums.

![Gradle signature verification report showing only validating checksums](/assets/2024-04-11-report.png){:width="600px"}

Generally, this happens when artifacts are missing their signature files (`.asc`), but this did not seem to be the case
for Guava.

![Guava Maven files for version 32.1.0-jre](/assets/2024-04-11-guava-maven.png){:width="600px"}

You, an eagle-eyed reader, might have already spotted something funky - we are using module
`com.google.guava:guava:32.1.3-jre`, but we are getting verification issues with `guava-32.1.3-android.jar`
which should only be used when `com.google.guava:guava:32.1.3-android` is requested.

*Note: Guava ships two "flavors" of their library: JRE and Android with various subtle differences*

We went back to see what's changed from Guava `32.0.1` and we noticed that now Guava ships with [Gradle module metadata](https://docs.gradle.org/current/userguide/publishing_gradle_module_metadata.html)
(`.module` files) that Gradle now uses to resolve dependencies instead of POMs. Modules files allow for [variant aware
resolution](https://docs.gradle.org/current/userguide/variant_model.html#understanding-variant-selection) which is
exactly what Guava started using. The relevant bit of `.module` file is the following:

```
    {
      "name": "androidRuntimeElements",
      "attributes": {
        "org.gradle.category": "library",
        "org.gradle.dependency.bundling": "external",
        "org.gradle.jvm.version": "8",
        "org.gradle.jvm.environment": "android",
        "org.gradle.libraryelements": "jar",
        "org.gradle.usage": "java-runtime"
      },
    ...
      "files": [
        {
          "name": "guava-32.1.3-android.jar",
          "url": "../32.1.3-android/guava-32.1.3-android.jar"
        }
      ],

    }
```

What this means, is that if you add a dependency on `com.google.guava:guava:32.1.3-jre` (directly or transitively) from
an Android project, it will actually resolve to `guava-32.1.3-android.jar` as Android Gradle Plugin will add
`"org.gradle.jvm.environment": "android"` attribute during resolution. This means instead of getting the `-jre.jar`
in your APK which is likely not compatible with Android you just get the right jar. There is a reverse set up in
`com.google.guava:guava:32.1.3-android` to get the `-jre.jar` for Java consumers. This all sounds awesome!

Wait, why are we getting no signature verification?! Turns out that `"files"` trickery with relative `url` makes Gradle
resolve the right jar file, but when it tries to verify this, it looks for
`com/google/guava/guava/32.1.3-jre/guava-32.1.3-android.jar.asc` that does not exist! Luckily, Gradle provides
`available-at` to handle this type of redirect and [now there is a Github issue](https://github.com/google/guava/issues/7154)
for Guava to migrate to that instead so that future versions not only get the right jar variant, but can also verify
these jars are signed as expected. Phew 😅
