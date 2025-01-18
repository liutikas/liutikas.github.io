---
layout: post
title:  Together in Isolation - Isolated Project Safe Way to Aggregate Optional Artifacts 
header: header-eng
mastodon: 113636641240393878
---

One common thing to do in a Gradle project is collect artifacts from a subset of projects to produce an aggregate
artifact. An example of it might be collecting test reports from each project that have tests to create a combined
test report for all projects. Historically, someone might have done this with:

Root project `build.gradle.kts`:
```kotlin
abstract class AggregatingTask : DefaultTask() {
    @get:InputFiles
    abstract val inputs: ListProperty<File>
    @get:OutputFile
    abstract val output: RegularFile
    @TaskAction fun aggregate() { /* process inputs */ }
}

tasks.register<AggregatingTask>("aggregatingTask")
```

Each project that has an artifact in their `build.gradle.kts`:
```kotlin
rootProject.tasks.named<AggregatingTask>(
    "aggregatingTask"
).configure { 
    inputs.add(reportTask.flatmap { it.reportFile })
}
```

Sadly, in addition to being fragile with hardcoding of the task name/type, it is also not valid with Gradle
[Isolated Project](https://docs.gradle.org/current/userguide/isolated_projects.html) feature enabled. Is there a better
way? Yes (but it is a tad verbose)!

Root project `build.gradle.kts`:
```kotlin
val reportConsumer = configurations.create("reportConsumer") {
    isCanBeResolved = true
    isCanBeConsumed = false
    attributes {
        attribute(
            Category.CATEGORY_ATTRIBUTE,
            objects.named("my-reports")
        )
    }
}
// Add all subprojects to dependencies
subprojects {
    reportConsumer.dependencies.add(dependencies.create(this))
}
// Be lenient if not all projects have this artifact
val reportCollection = reportConsumer.incoming.artifactView {
    lenient(true)
}.files

abstract class AggregatingTask : DefaultTask() {
    @get:InputFiles
    abstract val inputs: ConfigurableFileCollection
    @get:OutputFile
    abstract val output: RegularFile
    @TaskAction fun aggregate() { /* process inputs */ }
}
tasks.register<AggregatingTask>("aggregatingTask") {
    inputs.from(reportCollection)
}

```

Each project that has an artifact in their `build.gradle.kts`:
```kotlin
configurations.create("reportPublisher") {
    // Note, consumed and resolved are opposite of root project
    isCanBeResolved = false
    isCanBeConsumed = true
    attributes {
        attribute(
            Category.CATEGORY_ATTRIBUTE,
            objects.named("my-reports")
        )
    }
}
artifacts.add("reportPublisher", reportTask)
```

With this set up, root project still has the aggregating task that is able to build the report from all the artifacts
from each project, but instead of each subproject contributing to that task directly, everything flows through Gradle
artifact publishing and attribute resolution mechanisms.

Pro-tip: to visualize what you have added to artifacts, you can use `./gradlew :projectA:outgoingVariants`
