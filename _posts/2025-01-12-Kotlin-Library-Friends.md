---
layout: post
title:  Kotlin Library Friends - Using the Internals
header: header-eng
---

Let's say there is a library out there that has a very useful `internal` method inside of it.

```kotlin
package org.secret

internal class SecretSauce {
    fun onlyForFriends(): Int = 0
}
```

If you try to use it
```kotlin
import org.secret.SecretSauce
fun myUser(): Boolean {
    SecretSauce().onlyForFriends()
    return true
}
```

Kotlin compilation will fail with
```
> Task :lib:compileKotlin FAILED
e: file:///.../Library.kt:5:19 Cannot access 'class SecretSauce : Any': it is internal in file.
e: file:///.../Library.kt:8:5 Cannot access 'class SecretSauce : Any': it is internal in file.
e: file:///.../Library.kt:8:19 Cannot access 'class SecretSauce : Any': it is internal in file.
```

That's good! Library internal bits should not accessible be to you. However, we can bypass these errors with
```kotlin
@file:Suppress("INVISIBLE_MEMBER", "INVISIBLE_REFERENCE")
```
Jesse Wilson has [a great write-up about this suppression](https://publicobject.com/2024/01/30/internal-visibility/).

Sadly, [this workaround is getting phased out](https://youtrack.jetbrains.com/issue/KT-60866) starting with
Kotlin `2.1`, and now you will get a warning that will eventually be an insuppressible error.

Is there a different way?

One might wonder, how does `src/main` and `src/test` compilations work, as they are two different Kotlin invocations in
Gradle. If you look [at the compiler argument options](https://github.com/JetBrains/kotlin/blob/2.1.0/compiler/cli/cli-common/src/org/jetbrains/kotlin/cli/common/arguments/K2JVMCompilerArguments.kt#L520)
you'll find `-Xfriend-paths=path/to/classes/` that is used for `src/test` to be able to access `src/main` `internal`
types. If we were naughty, we could use this same mechanism for ourselves with external libraries.

```kotlin
// Create a configuration we can use to track friend libraries
val friends = configurations.create("friends") {
    isCanBeResolved = true
    isCanBeConsumed = false
    isTransitive = false
}
// Make sure friends libraries are on the classpath
configurations.findByName("implementation")?.extendsFrom(friends)
// Make these libraries friends :) 
tasks.withType<KotlinJvmCompile>().configureEach {
    val friendCollection = friends.incoming.artifactView { }.files
    compilerOptions.freeCompilerArgs.add(
        provider { "-Xfriend-paths=${friendCollection.joinToString(",")}"}
    )
}
// Add libraries you want to 
dependencies {
    friends("org.example:secret-sauce:1.0.0")
}
```

*Note*, any use of `internal` types is completely at your own risk, they are hidden on purpose and thus you might break
the library or the library owners will break you on the library update.
