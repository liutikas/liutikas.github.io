---
layout: post
title:  Trust But Verify - Enabling Gradle Signature Verification for An Android Project 
header: header-eng
---

In my post on [Gradle Security Considerations]({% post_url 2022-05-13-Gradle-Security-Considerations %}) I suggested
enabling [Gradle dependency signature verification](https://docs.gradle.org/current/userguide/dependency_verification.html#sec:signature-verification).
It is an important practice given today's landscape of build supply chain attacks where malicious actors attempt to ship
modified libraries masquerading as popular official libraries. My post left you without an explanation on how to enable
this verification for an Android project, so here goes that explanation.

To start you need an Android project that uses Gradle. In my case I will use [a simple Android project](https://github.com/liutikas/dependency-signature-verification-android/tree/ce78ee542546108e9ad0caa8c744f48f7a3c5ecc).

The first step is to add `gradle/verification-metadata.xml` that looks like:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<verification-metadata xmlns="https://schema.gradle.org/dependency-verification" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://schema.gradle.org/dependency-verification https://schema.gradle.org/dependency-verification/dependency-verification-1.3.xsd">
    <configuration>
        <!-- verify .pom and .module files -->
        <verify-metadata>true</verify-metadata>
        <!-- verify .asc PGP files that come with the artifacts -->
        <verify-signatures>true</verify-signatures>
        <!-- use human readable keyring format -->
        <keyring-format>armored</keyring-format>
        <!-- read keys in a local file, fewer requests to network -->
        <key-servers enabled="false">
            <key-server uri="https://keyserver.ubuntu.com"/>
            <key-server uri="https://keys.openpgp.org"/>
        </key-servers>
    </configuration>
    <components>
    </components>
</verification-metadata>
```

If you then try to run any Gradle command, e.g. `./gradlew assembleDebug`, it will fail with
```text
* What went wrong:
Error resolving plugin [id: 'com.android.application', version: '8.7.3', apply: false]
> Dependency verification failed for configuration 'detachedConfiguration1'
  One artifact failed verification: com.android.application.gradle.plugin-8.7.3.pom ...
  This can indicate that a dependency has been compromised ...
  
  Open this report for more details: .../dependency-verification-report.html
```

This is good - Gradle is telling you are pulling in dependencies that you have not approved!

The next step is to bootstrap the initial set trusted keys and components with:
```bash
./gradlew --write-verification-metadata pgp,sha256 --export-keys help
```

This command tells Gradle to build a list of PGP keys and fallback checksums for all the dependencies that
are used in this project. You will see a change to your `verification-metadata.xml` ([one for my example project](https://github.com/liutikas/dependency-signature-verification-android/blob/c897d0b45715f00f0017e23d0a3a547bd99f9845/gradle/verification-metadata.xml))
with a number of entries like:
```xml
<trusted-key id="8461EFA0E74ABAE010DE66994EB27DB2A3B88B8B">
    <trusting group="androidx.activity"/>
</trusted-key>
```
telling Gradle that if it sees a dependency from maven group `androidx.activity`, it will ensure that the accompanying
`.asc` files match that key.

You will also get `gradle/verification-keyring.keys` that actually contains the public PGP keys used by your build.
You should check in both of these files into your version tracking system. Any changes in the future that modify either
`verification-metadata.xml` or `verification-keyring.keys` should be carefully reviewed.

You could stop here, but there are a few optional steps that could make this even cleaner.

1. Strip `version="1.0.0"` attributes from `<trusted-key>` entries to ensure that you don't have to change these entries
when you update a dependency. Android Studio replace tool with regex from `<trusted-key(.*) version=\".*\"/>` to
`<trusted-key$1/>` will do it. ([change in my example project](https://github.com/liutikas/dependency-signature-verification-android/commit/b119f8ccf066b264cf4fe3e3c96206a17b93a555))
2. Looking at the list of `<component>` entries, you can upgrade to a version of a dependency that does have signatures
and then re-run `./gradlew --write-verification-metadata pgp,sha256 --export-keys help` which will remove no longer
needed entries ([change in my example project](https://github.com/liutikas/dependency-signature-verification-android/commit/9a0449b6b683d7f2ccfb6aaaef986fa3aef0559b)).
This will make sure that you are less vulnerable with that dependency
3. File bugs to projects that do not sign their artifacts or pull in dependencies that are not signed
(for example [Android Gradle Plugin pulling in `net.sf.kxml`](https://issuetracker.google.com/issues/294916131))

Happy verifying!
