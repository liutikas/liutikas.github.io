---
layout: post
title:  Treasure Cache - Gradle Cache Entries
header: header-eng
---

You are likely already using Gradle build cache (`org.gradle.caching=true`) and enjoying significant gains in productivity.
This post discusses what to do when things go wrong with a Gradle task and how build cache entries can aid in your investigations.

Let's say you have the following task:

```kotlin
@CacheableTask
abstract class MyTask: DefaultTask() {
    @get:Input abstract val inputString: Property<String>

    @get:OutputDirectory abstract val outputDirectory: DirectoryProperty
    @get:OutputFile abstract val outputFile: RegularFileProperty

    @TaskAction
    fun doWork() {
        outputDirectory.file("output.txt").get().asFile.writeText(
            inputString.get()
        )
        outputFile.get().asFile.writeText(
            inputString.get()
        )
    }
}

tasks.register<MyTask>("myTask") {
    inputString.set("Hello world")
    outputDirectory.set(layout.buildDirectory.dir("myTaskOutput"))
    outputFile.set(layout.buildDirectory.file("anotherOutput.txt"))
}
```

If you run `./gradlew lib:myTask`, delete your `lib/build` directory, and then run `./gradlew lib:myTask` again
you will see that the task outputs were restored from cache:

```text
BUILD SUCCESSFUL in 842ms
1 actionable task: 1 from cache
```

This is awesome, as it means you are skipping work! However, you might wonder how it works. Luckily, Gradle provides
a way for us to see how this works using the following command:

```bash
./gradlew lib:myTask --rerun -Dorg.gradle.caching.debug=true
```

This command will give you output like this:

```text
> Task :lib:myTask
Appending implementation to build cache key: Build_gradle$MyTask_Decorated@4cd7723d0f30d710dfb92ede78ac3f65
Appending additional implementation to build cache key: Build_gradle$MyTask_Decorated@4cd7723d0f30d710dfb92ede78ac3f65
Appending input value fingerprint for 'inputString' to build cache key: 098c453c6283a52d8b3ba9bf54f63b0b
Appending output property name to build cache key: outputDirectory
Appending output property name to build cache key: outputFile
Build cache key for task ':lib:myTask' is 631ae8841bc47d21c0087f54f7155af3
```

This output shows how Gradle computed the build cache key for this task. The more complex the inputs and outputs, the
more complex the computation. The build cache key `631ae8841bc47d21c0087f54f7155af3` is the important value for our needs.

This means we have a `~/.gradle/caches/build-cache-1/631ae8841bc47d21c0087f54f7155af3` cache entry for this task. If you
delete that file, you will see that Gradle has to rerun this task. This file is simply a `tar.gz` file, so if you rename
it to add the `.tar.gz` extension, you can open it in your favorite archive tool to see the contents:

```text
tree-outputDirectory
  outputFile.txt
tree-outputFile
METADATA
```

Inside, you'll find contents that look suspiciously familiar compared to our `lib/build` directory:

```text
myTaskOutput
  outputFile.txt
anotherOutput.txt
```

Specifically, `tree-outputDirectory/output.txt` (representing the `@get:OutputDirectory abstract val outputDirectory: DirectoryProperty`) is identical to
`lib/build/myTaskOutput/output.txt`, and `tree-outputFile` (representing the `@get:OutputFile abstract val outputFile: RegularFileProperty`) is identical to
`lib/build/anotherOutput.txt`.

There is also `METADATA` that can be inspected using the following code:

```kotlin
import org.gradle.caching.internal.origin.OriginMetadataFactory
import java.io.FileInputStream
import java.util.*

val invocationId = UUID.randomUUID().toString()
val originReader = OriginMetadataFactory(invocationId) { }.createReader()
val originMetadata = originReader.execute(FileInputStream("METADATA"))
println("""
    Cache key: ${originMetadata.buildCacheKey}
    Execution time: ${originMetadata.executionTime}
""")
```

```text
Cache key: 631ae8841bc47d21c0087f54f7155af3
Execution time: PT0.002S
```

Hopefully, this demystifies some details about how the Gradle cache works.

Using this method, we investigated [a task in the androidx build that was giving a different/non-deterministic output](https://issuetracker.google.com/issues/408463628).
We saw there that an annotation processor was generating code that kept changing order due to an assumption that `HashMap`
has a stable key iteration order.

![A fix for AppSearch code generator](/assets/2025-04-14-generator-fix.png){:width="800px"}
