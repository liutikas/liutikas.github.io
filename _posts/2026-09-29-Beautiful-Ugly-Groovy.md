---
layout: post
title: Beautiful Ugly Groovy
header: header-eng
mastodon: 117355504234355054
---

Let's say in your `build.gradle` scripts you want to allow users to set some values from a preset list

```groovy
androidx {
  mavenVersion = LibraryVersions.HILT
}
```

This can be achieved by putting the following in your `buildSrc` (or your included plugin build):

```kotlin
open class AndroidXExtension {
  fun setMavenVersion(version: String)
}
object LibraryVersions {
  const val HILT = "1.0.0"
}
```

`LibraryVersions` will need to be in a Kotlin file that doesn't specify the `package` as otherwise it will require
`import my.plugins.LibraryVersions` in the `build.gradle` file.

This solution will work, you will have an autocomplete. However, any change to `object LibraryVersions` will invalidate
build cache for all tasks in the project. That's a bummer. Is there a beautiful ugly way to avoid invalidations?

Turns out there is!

```kotlin
val map = project.providers.fileContents(
  project.rootProject.layout.projectDirectory.file("version.properties")
).asText.map {
  it.lines().associate { line ->
    val split = line.split('=', limit = 2)
    split[0] to split[1]
  }
}
project.extensions.create(
  "androidx",
  AndroidXExtension::class.java,
  map.get()
)
```

```kotlin
open class AndroidXExtension(
  @JvmField val LibraryVersions: Map<String, String>
) {
  fun setMavenVersion(version: String) {}
}
```

What this does is that makes
```groovy
androidx {
  mavenVersion = LibraryVersions.HILT
}
```
into a call
```kotlin
AndroidXExtension.setMavenVersion(AndroidXExtension.getLibraryVersions()["HILT"])
```

as we get the benefit of groovy magic of allowing map access that looks like a constant.

We loose autocomplete and invalid keys will only fail at runtime, but instead we gain no build cache
for build scripts invalidation on changes to `version.properties`. Only the Gradle configuration
needs to be re-run.

Side-note, you might ask "why not use Gradle version catalogs?" and the answer is that you might need much richer types
than the simplest example I used here, e.g. if the version includes additional metadata.

I am both proud and ashamed of my creation.