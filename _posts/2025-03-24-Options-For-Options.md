---
layout: post
title:  Options for Options - Ways to Pass Arguments to Gradle
header: header-eng
mastodon: 114220254831812831
---

When setting up your build, you might need to pass arguments to the Gradle, such as binary toggles like `"release-build=true"`
or strings values like `"my-key=my-value"`. As usual, Gradle offers more than one way to achieve this,
and this post will guide you through some of these options.

**Using Environment Variables**

```bash
MY_KEY=MY_VALUE ./gradlew myTaskWithEnvVar
```

and then read the value using a provider:

```kotlin
abstract class MyTaskWithEnvVar : DefaultTask() {
    @get:Input
    abstract val myKey: Property<String>

    @TaskAction
    fun doThings() { println(myKey.get()) }
}

tasks.register("myTaskWithEnvVar", MyTaskWithEnvVar::class.java) {
    myKey.set(providers.environmentVariable("MY_KEY"))
}
```

**Using Gradle Properties**

```bash
./gradlew myTask -PmyKey=MY_VALUE
```

and then read the value using a provider:

```kotlin
providers.gradleProperty("myKey")
```

**Caution**: Avoid using this one for values that frequently. Due to [a Gradle bug any Gradle property change will invalidate
configuration cache](https://github.com/gradle/gradle/issues/20969), even if that value is not used / read during
configuration.

**Using Task Options**

If your argument is specific to a single task, you can use [@Option](https://docs.gradle.org/nightly/javadoc/org/gradle/api/tasks/options/Option.html)
annotation on in input to your task.

```bash
./gradlew myTaskWithOptions --my-key=MY_VALUE
```

and then have it connected like this:

```kotlin
abstract class MyTaskWithOptions : DefaultTask() {
    @get:Input
    @get:Option(option = "my-key", description = "Very important value")
    abstract val myKey: Property<String>

    @TaskAction
    fun doThings() { println(myKey.get()) }
}
tasks.register("myTaskWithOptions", MyTaskWithOptions::class.java)
```

Gradle will wire up input for you.

**Using Files**

You might be familiar with `gradle.properties`, `local.properties`, `gradle/libs.versions.toml` and other files that
Gradle has defined for you. You can create your own and use them as inputs! You could use any file format you'd like,
but I'll give an example of using a Java properties file.

```bash
./gradlew myTask
```

The code to read the properties can look like this:

```kotlin
import java.util.Properties
abstract class MyTask4 : DefaultTask() {
    @get:InputFile
    abstract val myValues: RegularFileProperty

    @TaskAction
    fun doThings() {
        val props = Properties()
        props.load(myValues.get().asFile.inputStream())
        println(props["myKey"])
    }
}

tasks.register("myTask4", MyTask4::class.java) {
    myValues.set(layout.projectDirectory.file("my-values.properties"))
}
```

where `my-values.properties` has 
```properties
myKey=myValue
```

**Other considerations**

All the methods discussed, except for Gradle Properties, will play nicely with configuration cache. Consequently, modifying
the values will not invalidate configuration cache.

While you might also encounter [init scripts](https://docs.gradle.org/current/userguide/init_scripts.html) for setting values,
they are generally not recommended due to various issues, including breaking configuration cache on their change.
