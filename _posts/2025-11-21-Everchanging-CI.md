---
layout: post
title:  Everchanging CI - Isolating Configuration Cache Inputs
header: header-eng
mastodon: 115590717015050583
---

In the post [Input to Your Inputs]({% post_url 2025-09-30-Input-to-Your-Inputs %}) we discussed reducing inputs to the
configuration cache (CC) to increase your CC hit rate. One scenario that we had to tackle in AndroidX recently was
an environment variable that passes in a path to a file, and we want to use a portion of that file as an input to a
task. We do not care if the path to the file changes, we only want CC to be invalidated if the extracted portion of the
file changes.

To make it more concrete, here is an example invocation:

```bash
BUILD_INFO=/path/to/BUILD_ID_123.properties ./gradlew build
```

`BUILD_ID_100.properties` name changes with every build as it contains a unique build ID. The contents of this file
could look something like:

```properties
flavorOfTheDay=chocolate
otherInfo=noImportant
```

And finally, we want to pass this information into a task `FlavorGenerator` that has `flavorOfTheday: Property<String>`

A naive solution:

```kotlin
val flavorOfTheDayValue = System.getenv("BUILD_INFO")?.let { filePath ->
    val properties = Properties()
    properties.load(File(filePath).inputStream())
    properties.getProperty("flavorOfTheDay")
} ?: "fallbackFlavor"

tasks.register<FlavorGenerator>("generator") {
    flavorOfTheDay.set(flavorOfTheDayValue)
}
```

This will mostly function as expected, if you change the contents of the file pointed to by `BUILD_INFO` the task will rerun.
Sadly, this with the following issues:
- We are doing I/O during configuration phase, it should be avoided
  - Configuration happens even if the task in question is not run, thus we waste time
- Changing the path pointed to by `BUILD_INFO` will invalidate configuration
  - Call to `System.getenv("BUILD_INFO")` makes Gradle track the value of this environment variable as input to CC.
- Changing the contents of the file pointed to by `BUILD_INFO` will invalidate configuration.
  - Call to `File(filePath).inputStream()` makes Gradle track this file as input to CC.

A much better solution is to use providers:

```kotlin
val flavorOfTheDayProvider = providers.environmentVariable("BUILD_INFO").map { filePath ->
    val properties = Properties()
    properties.load(File(filePath).inputStream())
    properties.getProperty("flavorOfTheDay")
}.orElse("fallbackFlavor")

tasks.register<FlavorGenerator>("generator") {
    flavorOfTheDay.set(flavorOfTheDayProvider)
}
```

While this solution looks nearly identical to the previous one, in this set up with have CC hits for both the changes
to the path, and the contents of the file changing. The only thing that gets invalidated in the task if the
`flavorOfTheDay` has in fact changed. Even better, even removing `BUILD_INFO` has a CC hit. Gradle is able to do this
because we use providers both on the task, as well when creating the input.

```bash
$ BUILD_INFO=./lib/BUILD_ID_100.properties ./gradlew generator
Calculating task graph as no cached configuration is available for tasks: generator
...
$ BUILD_INFO=./lib/BUILD_ID_101.properties ./gradlew generator
Reusing configuration cache.
...
$ ./gradlew generator
Reusing configuration cache.
...
```

For more complex inputs, you can use [ValueSource](https://docs.gradle.org/nightly/javadoc/org/gradle/api/provider/ValueSource.html)
that allows you for example to call git to get HEAD SHA

```kotlin
abstract class GitShaValue: ValueSource<String, GitShaValue.Parameters> {
    interface Parameters : ValueSourceParameters {
        val workingDir: DirectoryProperty
    }
    @get:Inject abstract val execOperations: ExecOperations
    override fun obtain(): String? {
        val output = ByteArrayOutputStream()
        execOperations.exec {
            commandLine("git", "rev-parse", "HEAD")
            standardOutput = output
            workingDir = parameters.workingDir.get().asFile
        }
        return String(output.toByteArray()).trim()
    }
}

val gitShaProvider = providers.of(GitShaValue::class.java) {
    parameters.workingDir.set(rootDir)
}
```

This will make sure to only invalidate your CC when the HEAD SHA changes.

Note, Gradle potentially needs to recalculate all the providers before it decides to reuse CC. This is especially true
for `ValueSource`. If your `ValueSource` executes a slow command even your CC hits will be slow, so you need to be very
careful with the use of `ValueSource`.
