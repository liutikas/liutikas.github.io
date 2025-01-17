---
layout: post
title:  Wrapper's delight - Gradle wrapper task uses
header: header-eng
---

Gradle has a built-in task called `wrapper` that `./gradlew tasks` describes with:

```text
wrapper - Generates Gradle wrapper files.
```

Not terribly descriptive, but it is quite a useful task for the following cases:

**Updating to a different version of Gradle**

```bash
./gradlew wrapper --gradle-version 8.12
```

This will update your `gradle/gradle/gradle-wrapper.properties` to that version of Gradle. It also validates that
such version exists.

Note, you can also specify `latest`, `release-candidate`, `nightly`, or `release-nightly`, for example:

```bash
./gradlew wrapper --gradle-version latest
```

**Moving between `all` and `bin` distributions**

```bash
./gradlew wrapper --distribution-type bin
```

This will move you away from [a giant zip to a large zip]({% post_url 2024-12-18-What-The-Distribution %}).

**Replacing existing wrapper with one you trust**

As noted in my post on [Gradle security considerations]({% post_url 2022-05-13-Gradle-Security-Considerations %}) you
should generally replace the wrapper of any external project you clone as it might have some nasty surprises. Wrapper
task can help you do that using Gradle installation you already trust.

```bash
rm -fr gradle/wrapper/ gradlew*
path/to/trusted/gradlew wrapper --gradle-version latest 
```

That is all!