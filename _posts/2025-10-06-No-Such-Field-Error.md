---
layout: post
title:  NoSuchFieldError - JavaVersion does not have member field 'JavaVersion$Companion Companion'
header: header-eng
mastodon: 115326304729718007
---

AndroidX team strongly believes in [shadow jobs](https://slack.engineering/shadow-jobs/) to catch regressions early.
Metalava tool that we use to track androidx APIs builds on top of Android Lint infrastructure, which in turn builds on
top of Jetbrains' PSI/UAST. Given this reliance, we run a shadow build that tests Metalava against HEAD of Android Lint
for each commit to Android Lint project. On September 12th we got an alert that nearly every Metalava test started
failing with a stacktrace like:

```
java.lang.NoSuchFieldError: Class com.intellij.util.lang.JavaVersion does not have member field 'com.intellij.util.lang.JavaVersion$Companion Companion'
	at com.intellij.pom.java.LanguageLevel.<init>(LanguageLevel.kt:92)
	at com.intellij.pom.java.LanguageLevel.<clinit>(LanguageLevel.kt:31)
	at com.android.tools.metalava.model.psi.PsiEnvironmentManager$Companion.javaLanguageLevelFromString(PsiEnvironmentManager.kt:166)
	at com.android.tools.metalava.model.psi.PsiEnvironmentManager.createSourceParser(PsiEnvironmentManager.kt:132)
	at com.android.tools.metalava.model.source.EnvironmentManager$DefaultImpls.createSourceParser$default(EnvironmentManager.kt:41)
	at com.android.tools.metalava.model.source.SourceModelSuiteRunner.createTestCodebase(SourceModelSuiteRunner.kt:87)
	at com.android.tools.metalava.model.source.SourceModelSuiteRunner.createCodebaseAndRun(SourceModelSuiteRunner.kt:67)
	at com.android.tools.metalava.model.testsuite.BaseModelTest.createCodebaseFromInputSetAndRun(BaseModelTest.kt:235)
	at com.android.tools.metalava.model.testsuite.BaseModelTest.runCodebaseTest(BaseModelTest.kt:295)
	at com.android.tools.metalava.model.testsuite.BaseModelTest.runCodebaseTest$default(BaseModelTest.kt:288)
	at com.android.tools.metalava.model.psi.PsiAnnotationMixtureTest.runMixtureAnnotationTest(PsiAnnotationMixtureTest.kt:79)
	at com.android.tools.metalava.model.psi.PsiAnnotationMixtureTest.Test java usage, java definition(PsiAnnotationMixtureTest.kt:124)
	at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:103)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	...
```

It all started with the following changes:

```text
Rewrite IJ core to remove unwanted app disposal connection : platform/prebuilts/tools
Rewrite PluginStructureProvider to adapt to IJ 2025.2 : platform/prebuilts/tools
Update lint-psi artifacts : platform/prebuilts/tools
Update lint-psi to 252.23892.409 : platform/prebuilts/tools
Clarify the previous jar rewriter is for IntelliJ : platform/prebuilts/tools
Adapt to IntelliJ 2025.2 (standalone-render) : platform/tools/base
Adapt to IntelliJ 2025.2 (Lint) : platform/tools/base
Update Lint dependencies published to gmaven : platform/tools/base
```

The failing `LanguageLevel.kt:92` line is a `val level = LanguageLevel.parse(value)` call. A brief look at these commits
provided little insight on what was going wrong, but given the stacktrace it seemed that
there was a binary incompatible change to `com.intellij.util.lang.JavaVersion` class. I reached out to the author of
these changes, and he pointed me to [.java to .kt rename](https://github.com/JetBrains/intellij-community/commit/f3370984c94f91a9ed527104732f92d368f363f9)
and [conversion to Kotlin](https://github.com/JetBrains/intellij-community/commit/4bbedb572d09ca9d44ccc404ebb4bbd0e0cdf17b#diff-a0e8ddff3210614cc168436d6cbc919f99080e73e5b846997da481a3decb8d63)
changes in Jetbrains' repository. We can now see that `LanguageLevel.parse` moved from being a static Java
method to a companion method in Kotlin. This is, in fact, a binary breaking change, but it happens to be source compatible
for Kotlin callers. From the stacktrace we know it is calling the companion version, but it is missing that method,
which then we can reason that we get an old version of JavaVersion. Why are we getting the old version of this class?

First attempt was to try to upgrade to the latest version of all dependencies to see if that fixes the issue. Sadly,
this results in the same failure. No progress.

We want to know which artifacts do `LanguageLevel` and `JavaVersion` come from and a quick look in the IDE points to
both of them being in `com/android/tools/external/com-intellij/intellij-core/32.0.0-alpha08/intellij-core-32.0.0-alpha08.jar`.
This makes it extra confusing as we should then get the correct version, but we clearly do not!

My next hunch was that we are getting two copies of `JavaVersion.class` and to verify I need to know all the jars
involved. I wrote a short snippet to print out all the jars on the classpath
```kotlin
tasks.withType(Test::class.java).configureEach {
    doFirst { classpath.files.forEach { println(it.absolutePath) } }
}
```

The number of jars was simply too large check manually, so I went for AI to extend my snippet and got the following:
```kotlin
tasks.withType(Test::class.java).configureEach {
    doFirst {
        println("Searching for JARs containing 'com/intellij/util/lang/JavaVersion.class'...")
        classpath.files
            .filter { it.isFile && it.name.endsWith(".jar") }
            .forEach { jarFile ->
                try {
                    // Use Kotlin's `use` function for safe handling of the ZipFile resource
                    ZipFile(jarFile).use { zip ->
                        if (zip.getEntry("com/intellij/util/lang/JavaVersion.class") != null) {
                            println("Found in: ${jarFile.absolutePath}")
                        }
                    }
                } catch (e: ZipException) {
                    // This can happen with corrupted or non-standard JAR files
                    // You can log this if you want, but for this case, we'll ignore it.
                    // logger.warn("Could not read zip file: ${jarFile.path}", e)
                }
            }
    }
}
```

which gave me:
```text
> Task :metalava-model-psi:test
Searching for JARs containing 'com/intellij/util/lang/JavaVersion.class'...
Found in: .gradle/caches/modules-2/files-2.1/com.android.tools.external.com-intellij/kotlin-compiler/32.0.0-alpha08/9f76ba4930f2c91c62a05ef569a5f294a1b3c14c/kotlin-compiler-32.0.0-alpha08.jar
Found in: .gradle/caches/modules-2/files-2.1/com.android.tools.external.com-intellij/intellij-core/32.0.0-alpha08/9c0eb597408295ba28d6bb4f080d390c6a389ad0/intellij-core-32.0.0-alpha08.jar
```

Aha! Taking at look at these jars and I can see these classes present in both and at a different version. [We now have
our root cause](https://issuetracker.google.com/issues/449031505)!

I then modified the code to find all duplicate classes between these two jars and I found a dozen more potential issues.

The lesson learned here is that bundling (uber-jars) can result in obscure behavior if you are not very careful.