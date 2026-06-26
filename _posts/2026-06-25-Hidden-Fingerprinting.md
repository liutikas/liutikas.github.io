---
layout: post
title:  Hidden Fingerprinting - Hard to Measure Gradle Task Cost
header: header-eng
mastodon: 116813741317098512
---

Let's say you have a simple task that prints out current git commit:

```kotlin
abstract class MyTask : DefaultTask() {
    @get:Input abstract val gitSha: Property<String>
    @TaskAction fun doThings() {
        println(gitSha.get())
    }
}
```

and the use `ValueSource` as a mechanism to provide this value:

```kotlin
internal abstract class GitHeadShaSource : ValueSource<String, GitHeadShaSource.Parameters> {
    interface Parameters : ValueSourceParameters { val workingDir: DirectoryProperty }
    @get:Inject abstract val execOperations: ExecOperations
    override fun obtain(): String {
        val output = ByteArrayOutputStream()
        execOperations.exec {
            commandLine("git", "rev-parse", "HEAD")
            standardOutput = output
            workingDir = parameters.workingDir.get().asFile
        }
        return String(output.toByteArray(), Charset.defaultCharset()).trim()
    }
}
```

finally wiring this task like:

```kotlin
tasks.register("myTask", MyTask::class.java) {
    gitSha.set(
        providers.of(GitHeadShaSource::class.java) {
            parameters.workingDir.set(project.rootDir)
        }
    )
}
```

This post was inspired by my investigation of an almost identical setup in AndroidX has for generating some per project
metadata files. I was seeing an unexpectedly high duration for something that should only take milliseconds.

![Build Scan Task Details](/assets/2026-06-25-slowtask.png)

Measuring `@TaskAction` time was only making milliseconds as expected, however the duration was >1 second when running
many instances of this task across different projects at the same time. The breakthrough was to see that the build scan
was showing a very high "fingerprinting inputs" time:

![Build Scan Task Duration Details](/assets/2026-06-25-durationdetails.png)

`Fingerprinting inputs` step is the step where Gradle *serially* checks what is the current value of each task input.
This means executing the `ValueSource.obtain()` methods to know whether a task is up to. If you call that on hundreds of
individual provider instances for each project that produces a huge I/O pressure as `git` tries to return the commit SHA.

**Note**: this step happens every time you run that task in Gradle, even when you have a configuration cache hit.

The key is that Gradle does this work for each *instance* of a provider, so if a provider is shared, it only needs to
run it once. For example, if you have multiple tasks within the same project, you can do the following:

```kotlin
val gitShaProvider = providers.of(GitHeadShaSource::class.java) {
    parameters.workingDir.set(project.rootDir)
}

tasks.register("myTask1", MyTask::class.java) {
    gitSha.set(gitShaProvider)
}
tasks.register("myTask2", MyTask::class.java) {
    gitSha.set(gitShaProvider)
}
```

This will only run `obtain()` once when trying to run both tasks.

If you need something shared across multiple projects, you can use `BuildService`:

```kotlin
abstract class SharedProviderService : BuildService<SharedProviderService.Parameters> {
    interface Parameters : BuildServiceParameters {
        var gitShaProvider: Provider<String>
    }
    fun getGitShaProvider(): Provider<String> = parameters.gitShaProvider
    companion object {
        internal fun registerOrGet(project: Project): SharedProviderService {
            return project.gradle.sharedServices.registerIfAbsent(
                "sharedProviderService",
                SharedProviderService::class.java,
            ) { spec ->
                spec.parameters.gitShaProvider = project.providers.of(GitHeadShaSource::class.java) {
                    parameters.workingDir.set(project.rootDir)
                }
            }.get()
        }
    }
}
```

Then you can do:

```kotlin
val gitShaProvider = SharedProviderService.registerOrGet(project).getGitShaProvider()
tasks.register("myTask1", MyTask::class.java) {
    gitSha.set(gitShaProvider)
}
```

In androidx case, this sped up `./gradlew createLibraryBuildInfoFiles` by 24x (3.45s -> 0.14s).

Happy provider sharing!