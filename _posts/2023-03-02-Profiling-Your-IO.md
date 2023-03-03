---
layout: post
title:  Profiling your I/O
header: header-eng
---

In [Profiling - The Good Kind]({% post_url 2022-06-01-Profiling-The-Good-Kind %})
and [Gradle Configuration - Death by a Thousand Cuts]({% post_url 2022-06-02-Death-By-A-Thousand-Cuts %})
I talked about importance of every ms in Gradle configuration phase. Reducing
your I/O in this phase is another important aspect. In this post I share how
I stumbled into and fixed unnecessary I/O reads in AndroidX Gradle configuration
phase.

As I looked into a trace of `./gradlew build --dry-run` in a clean build cold
daemon case I found that in the first 20 seconds there is a surprisingly long
2.6 seconds spent in `PGPPublicKeyRing`.

![Screenshot of CPU usage in first 20 seconds](/assets/2023-03-02-first-20s.png){:height="500px"}

AndroidX runs with [Gradle signature verification enabled](https://docs.gradle.org/current/userguide/dependency_verification.html#sec:signature-verification)
and so it wasn't surprising that it was reading the file, but it was surprising
that it was taking this much time for a 760kb file from an SSD. Seeing that this
was a file read event, I jumped to the events tab to see what was happening.

![Screenshot of File read events log](/assets/2023-03-02-file-events.png){:height="230px"}

Here I found tens of thousands of reads of `verification-keyrings.keys`, each
one of them a 1 byte read. Then I clicked on one of the entries to see a
stacktrace.

![Screenshot of File read events log](/assets/2023-03-02-stacktrace.png){:height="300px"}

Looking at this, it was either a bug in BouncyCastle PGP library or Gradle.
In [SecuritySupport.java:111](https://cs.android.com/android-studio/gradle/+/master:subprojects/security/src/main/java/org/gradle/security/internal/SecuritySupport.java;l=108;bpv=1;bpt=0;drc=5df54be90e47032ae48017232e1be0bd6ff8d95b)
we can see that [`createInputStreamFor(keyringFile)`](https://cs.android.com/android-studio/gradle/+/master:subprojects/security/src/main/java/org/gradle/security/internal/SecuritySupport.java;l=124;drc=5df54be90e47032ae48017232e1be0bd6ff8d95b;bpv=1;bpt=0) is used to create this
input stream sent to BouncyCastle library.

```java
private static InputStream createInputStreamFor(
    File keyringFile) throws IOException {
        InputStream stream = new FileInputStream(keyringFile);
        if (keyringFile.getName().endsWith(KEYS_FILE_EXT)) {
            return new ArmoredInputStream(stream);
        }
        return stream;
    }
```

Using `FileInputStream` without wrapping it `BufferedInputStream` results into
an I/O call for every `InputStream#read()` call. That was [exactly the fix
for Gradle](https://github.com/gradle/gradle/pull/24105)! We'll have a
fix for this in Gradle 8.1.

Using the same process, I was also able to find that AndroidX buildSrc code was
doing I/O during configuration phase to check if a given project has any test
code and [got to remove it](https://r.android.com/2460054)
saving us hundreds of I/O calls in this phase.
