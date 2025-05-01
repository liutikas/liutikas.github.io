---
layout: post
title:  Escape (de)Velocity - Artisanal Debugging of Tasks
header: header-eng
---

Debugging Gradle tasks can be challenging, especially when you have no access to tools like Develocity or need to work
offline. This post shares a couple of strategies to help you gain more insight into your Gradle build.

## Detailed Task Information

Knowing which tasks run for a given Gradle command is often the first step in debugging. Gradle provides a built-in
mechanism for this: `./gradlew foo:bar --dry-run`. This command lists all tasks that *would* run without actually
executing them. However, this only provides task names, not the types, inputs, outputs, or dependencies.

To get this richer information, you can add the following snippet to your build:

```kotlin
fun Project.setUpTaskDetails() {
    tasks.configureEach { task ->
        // Store dependencies in a provider to ensure they are resolved lazily
        // and can be safely read in doFirst/doLast.
        val deps = project.provider { task.taskDependencies.getDependencies(task).map { it.path } }
        task.doFirst {
            println("""
name: ${task.path}
type: ${task::class.java.superclass}
inputs:
${task.inputs.files.joinToString(prefix=" - ", separator = "\n - ")}
sourcefiles:
${task.inputs.sourceFiles.joinToString(prefix=" - ", separator = "\n - ")}
outputs:
${task.outputs.files.joinToString(prefix=" - ", separator = "\n - ")}
dependencies:
${deps.get().joinToString(prefix=" - ", separator = "\n - ")}
    """)
        }
    }
}
```

Now, when you run a Gradle command, this snippet will print out detailed information for each task that executes.

Example output:

```text
name: :lib:jar
type: class org.gradle.api.tasks.bundling.Jar
inputs:
 - /tasks-debug/lib/build/classes/kotlin/main/META-INF/lib.kotlin_module
 - /tasks-debug/lib/build/classes/kotlin/main/org/example/Library.class
 - /tasks-debug/lib/build/tmp/jar/MANIFEST.MF
outputs:
 - /tasks-debug/lib/build/libs/lib.jar
dependencies:
 - :lib:classes
 - :lib:compileKotlin
 - :lib:compileJava
```

## Detecting Shared Task Outputs

Non-reproducible builds can sometimes occur when multiple tasks write to the same output directory or file. If the
execution order of these tasks changes, you might get different results. It is best to ensure that each task has exclusive
ownership of its output directories/files.

Here's a custom task to help identify such issues:

```kotlin
abstract class TaskOutputTracker : DefaultTask() {
    init {
        project.gradle.taskGraph.whenReady {
            // This is eagerly initializing all tasks and thus is evil
            project.tasks.all { task ->
                allTaskOutputs.put(task.path, task.outputs.files.map { it.absolutePath })
            }
        }
    }

    @get:Input
    val allTaskOutputs: MapProperty<String, List<String>> =
        project.objects.mapProperty(String::class.java, List::class.java as Class<List<String>>)

    @TaskAction
    fun validate() {
        val pathToTask: MutableMap<String, MutableSet<String>> = mutableMapOf()
        allTaskOutputs.get().forEach { (taskPath, outputFiles) ->
            outputFiles.forEach { outputFile ->
                val set = pathToTask.getOrDefault(outputFile, mutableSetOf())
                set.add(taskPath)
                pathToTask[outputFile] = set
            }
        }
        val badTasks = pathToTask.filter { (_, taskPaths) -> taskPaths.size > 1 }
        if (badTasks.isNotEmpty()) {
            badTasks.forEach { (outputFile, taskPaths) ->
                println("$outputFile -> ${taskPaths.joinToString(", ")}")
            }
            throw Exception("You've got tasks with duplicate output directories")
        }
    }
}
```

An example of task output:

```text
/tasks-debug/lib/build/foo/lib.zip -> :lib:zip1, :lib:zip2

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':lib:taskOutputTracker'.
> java.lang.Exception: You've got tasks with duplicate output directories
```

## Identifying Eagerly Created Tasks

For optimal build performance and configuration times, Gradle tasks should be [configured lazily](https://docs.gradle.org/current/userguide/lazy_configuration.html).
This means tasks are only fully configured if they are actually needed for a requested build.

To find tasks that are being created eagerly even though they are not necessary, you can use the following snippet:

```kotlin
// register a task that has no dependencies
tasks.register("lonelyTask")

// utilize the fact that configureEach actions are only called
// if the task gets created
tasks.configureEach {
    if (name !in listOf(
            // the task we expect to be created
            "lonelyTask", 
            // tasks that Gradle 8.14 creates, but they shouldn't 
            "help",
            "projects",
            "tasks",
            "properties",
            "dependencyInsight",
            "dependencies",
            "buildEnvironment",
            "components",
            "model",
            "clean",
            "dependentComponents",
            "outgoingVariants",
            "resolvableConfigurations",
    )) println("Eagerly created task $path")
}
```

Running `./gradlew lonelyTask` will give you a list of tasks be created. It will look something like:

```text
Eagerly created task :lib:test
Eagerly created task :lib:jar
```

Good luck with your Gradle debugging!
: