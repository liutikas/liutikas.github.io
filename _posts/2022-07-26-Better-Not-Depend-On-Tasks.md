---
layout: post
title:  Better Not Depend On Tasks - That's the Old Timey Way
header: header-eng
---

In Gradle it is very common to have one task write to a file, and then that file
is read by another task to do more work. A couple of years back these tasks would
have looked something like:
```kotlin
open class OldWriter : DefaultTask() {
  @get:Input
  val list: MutableList<String> = mutableListOf()
  @get:OutputFile
  lateinit var outputFile: File
  @TaskAction
  fun write() = outputFile.writeText(list.joinToString("\n"))
}
open class OldReader : DefaultTask() {
  @get:InputFile
  lateinit var inputFile: File
  @get:OutputFile
  lateinit var outputFile: File
  @TaskAction
  fun writeCount() = outputFile.writeText(
    "Line count: ${inputFile.readLines().count()}"
  )
}
```
We would then set these tasks up using something like:
```kotlin

val oldWriterFile = project.buildDir.resolve("oldWriter.txt")
val oldWriterTask = tasks.register(
  "oldWriter",
  OldWriter::class
) {
  list.addAll(listOf("apple", "orange", "banana"))
  outputFile = oldWriterFile
}
tasks.register("oldReader", OldReader::class) {
  inputFile = oldWriterFile
  dependsOn(oldWriterTask)
  outputFile = project.buildDir.resolve("oldReader.txt")
}
```
This set up has an assortment of potential fragile spots. For example,
`dependsOn(oldWriterTask)` is load-bearing, and thus if accidentally deleted in
a refactor, it would make it so that `OldReader.inputFile` file would sometimes
be missing depending on what tasks your user invokes. We also have to rely
on Kotlin's `lateinit` for the files to make them non-null and hope that
task configuration is handled correctly.

Luckily, Gradle now has a better way to do this! Here is what tasks can look like:
```kotlin
abstract class NewWriter : DefaultTask() {
  @get:Input
  abstract val list: ListProperty<String>
  @get:OutputFile
  abstract val outputFile: RegularFileProperty
  @TaskAction
  fun write() = outputFile.get().asFile.writeText(
    list.get().joinToString("\n")
  )
}
abstract class NewReader : DefaultTask() {
  @get:InputFile
  abstract val inputFile: RegularFileProperty
  @get:OutputFile
  abstract val outputFile: RegularFileProperty
  @TaskAction
  fun writeCount() = outputFile.get().asFile.writeText(
    "Line count ${inputFile.get().asFile.readLines().count()}"
  )
}
```
One neat thing to note here, we are making our tasks and properties `abstract`
so we can let Gradle initialize `RegularFileProperty`, and `ListProperty` for us.

To set configure these tasks we use:
```kotlin
val newWriterTask = tasks.register(
  "newWriter",
  NewWriter::class
) {
  list.addAll("apple", "orange", "banana")
  outputFile.set(layout.buildDirectory.file("newWriter.txt"))
}
tasks.register("newReader", NewReader::class) {
  inputFile.set(newWriterTask.flatMap { it.outputFile })
  outputFile.set(layout.buildDirectory.file("newReader.txt"))
}
```

Using this set up, we no longer need `dependsOn(newWriterTask)` explicit task
dependency. `newWriterTask.flatMap` does the task dependency connection for us,
so when it `flatMap` is called to resolve the `outputFile`, it will run the
task to create this file. Additionally, we move away from `lateinit` and the
pseudo-nullness that it brings.

Similarly to the file, `NewWriter` can also now lazily get `list` that it will
write out, you can just call `list.set(someProviderHere)`, potentially even
using `flatMap` as in the example above.

To summarize, any time you see `dependsOn` in your code, you should be eyeing it
with suspicion as there are more robust ways available to wire-up your tasks.
