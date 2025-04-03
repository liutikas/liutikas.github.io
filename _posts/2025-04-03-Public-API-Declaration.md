---
layout: post
title:  Public (API) Declaration
header: header-eng
---

It's helpful to think of libraries as having of two parts:
1. **API surface**: the set of classes, interfaces, and methods that users are intended to interact with.
2. **Implementation details**: the underlying code that makes the API surface work.

Visibility modifiers like `private`, `internal`, or package-private can be used to control which parts of your
library are exposed as the API surface.

The same concept of separating API from implementation applies to your library's dependencies. Some dependencies are
required for users to interact with your API surface (and thus become part of your transitive API), while others are only
needed as an implementation detail. Gradle plugins such as [`java-library`](https://docs.gradle.org/current/userguide/java_library_plugin.html),
and `com.android.library` support this distinction by allowing library authors to declare their dependencies using
either `api` or `implementation` configuration.

```kotlin
dependencies {
    api("com.part:of-api:1.0.0")
    implementation("com.internal:detail:1.0.0")
}
```

After following the steps above, when a user adds your library as a dependency in their project, they will only get
access to your API surface and the dependencies needed to use that API surface.

**Important Note:** Gradle itself does not verify that you correctly marked all of your API dependencies. It will allow
you to publish a library where a necessary API dependency was mistakenly declared as implementation. This it make
it impossible to use the part of your API surface that needs this dependency, forcing user to manually add that
dependency explicitly.

Gradle accomplishes this separation of API and implementation through Gradle metadata (`.module` files) that are
published alongside of your library artifacts. This metadata allows the user to get two distinct classpaths: compile-time vs runtime.
The metadata file contains a `variants` section, which details the different ways your library can be consumed, along with
their dependencies based on the [requested attributes](https://docs.gradle.org/current/userguide/variant_attributes.html).

A simplified example of a `.module` file might look like this:

```json
{
  "variants": [
    {
      "name": "apiElements",
      "attributes": { "org.gradle.usage": "java-api" },
      "dependencies": [
        { "group": "com.part", "module": "of-api", "version": { "requires": "1.0.0" } }
      ],
      "files": [ { "url": "lib-1.0.0.jar" } ]
    },
    {
      "name": "runtimeElements",
      "attributes": { "org.gradle.usage": "java-runtime" },
      "dependencies": [
        { "group": "com.part", "module": "of-api", "version": { "requires": "1.0.0" } },
        { "group": "com.internal", "module": "detail", "version": { "requires": "1.0.0"}}
      ],
      "files": [ { "url": "lib-1.0.0.jar" } ]
    }
  ]
}
```

The same underlying mechanism of variants is used to implement platform selection for Kotlin multiplatform artifacts.

However, it is not always possible to rely on the JVM visibility to define your API surface. For instance, code generation
tools might only produce public types, or certain tools interacting with your library might require implementation of
types to be public. In such scenarios, it is common (but imperfect) practice to place these JVM-public-but-not-API types
in Java packages containing `internal` (e.g. `com.example.internal`). The hope is that library users will avoid using these
types. Unfortunately, IDE features like auto-complete and automatic import insertions make accidental usage quite easy,
as seen with examples like `com.android.build.gradle.internal.tasks.factory.dependsOn` when working with Gradle and
Android Gradle Plugin.

One way to solve this is to add standalone `*-api` artifacts to allow users to have clean compile classpath. For example,
Android Gradle Plugin (AGP) has `com.android.tools.build:gradle-api`. Sadly, this is still too easy to make mistakes,
because if someone includes `com.android.tools.build:gradle`, we are back to having all the JVM-public-but-not-API types.

A more robust solution to prevent accidental usage of internal types at compile time is to leverage the variant mechanism
to *replace* the main library artifact (which contains both API and implementation) with a separate artifact that *only*
includes your intended API surface.

```kotlin
// Create new source set
val publicSourceSet = sourceSets.create("public")
// Allow sources in main to use types defined in public
sourceSets.named("main").configure {
    runtimeClasspath += publicSourceSet.output
    compileClasspath += publicSourceSet.output
}
// Register a task that produces a jar with API surface
val publicJar = tasks.register<Jar>("publicJar") {
    from(publicSourceSet.output)
    archiveClassifier.set("public")
}
// Clear the previous jar and replace it with the public API jar
configurations.named("apiElements").configure {
    outgoing.artifacts.clear()
    outgoing.artifact(publicJar)
}
```

With this set up, any code you place in `src/public/kotlin` source set will visible at compile-time for users of your library,
Code in your `src/main/kotlin` will only be available at runtime.

In fact, AGP settings plugin `com.android.settings` [already employs this pattern](https://dl.google.com/android/maven2/com/android/tools/build/gradle-settings/8.9.1/gradle-settings-8.9.1.module)
and only exposes the API surface by default, demonstrating the effectiveness of this approach. 
