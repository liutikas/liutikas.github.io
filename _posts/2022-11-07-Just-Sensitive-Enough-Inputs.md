---
layout: post
title:  Making Gradle Inputs Just Sensitive Enough
header: header-eng
---

Gradle wants all your tasks to have inputs and outputs annotated in order to
be able to correctly detect when these tasks need to be rerun. This makes a task
run again if your `@get:Input val goodNumber: Property<Int>` goes from 1 to 5.
File inputs (`@InputFile` and `@InputFile`) behave similarly, but sadly require
additional attention for optimal behavior. By default, Gradle treats all the file
inputs as if their absolute path is important, so it uses full `/usr/mary/path/`
path for task input key calculation. This means that moving your source to a
different directory `/usr/mary/path2` results in no local build cache hits,
and even more importantly no remote cache hits for anyone else on the team
unless they happen to have their in an identical location. You will want to
pair your input file annotations with `@PathSensitivity` or `@Classpath` to
work around this bad default. Pick them in the following order:
1. `@Classpath` - a jar type input and the task only cares about changes to the ABI of the jar.
2. `@PathSensitivity(NONE)` - only care about the contents of the file.
3. `@PathSensitivity(NAME_ONLY)` - same as above, but also care about the simple file name.
4. `@PathSensitivity(RELATIVE)` - same as above, but you also use relative input path in your task.
5. `@PathSensitivity(ABSOLUTE)` - this is almost always a bug or a poor task design

Gradle does warn you about this bad default [deep in their PathSensitivity docs](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/PathSensitivity.html#ABSOLUTE),
however it is hard to find. If you are writing your tasks in a project using `java-gradle-plugin`,
then I recommend enabling the following to help catch these unannotated inputs.
```groovy
validatePlugins {
    enableStricterValidation = true
}
```
