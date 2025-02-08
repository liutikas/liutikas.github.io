---
layout: post
title:  Tragedy of Build Classpath
header: header-eng
---

Gradle is a JVM application and Gradle Plugins are libraries that get loaded on the Gradle build classpath in order to
execute work inside the Gradle process. With this great power comes great responsibility. As Tony's post
[When POM files lie](https://dev.to/autonomousapps/this-is-why-we-cant-have-nice-things-when-pom-files-lie-3lm5) highlights
one of the dangers, I want to share some best practices for Gradle plugin development.

- **Design a clear DSL**: Expose a well-defined DSL for all configurable aspects of your plugin and keep your tasks hidden
(e.g. inside an internal Java package). Tasks are implementation details and should not clutter user's classpath.
- **Thoroughly document your DSL**: Comprehensive documentation is essential. Modern IDEs display documentation for DSL
methods within `build.gradle.kts` (and `build.gradle`) making it easy for users to understand and utilize your plugin.
- **Only apply a single Gradle extension**: multiple entry points in `build.gradle.kts` can lead to confusion and make
the plugin harder to use.
- **Restrict JVM visibility**: Use the lowest possible JVM visibility (private > package protected / internal > public)
for types, methods, and properties. Public visibility can lead to unintended and unsupported usage, which can be difficult
to remove without breaking your users.
- **Minimize dependencies**: Avoid adding library dependencies. Instead of including an entire library like Guava for a use
of a single class, consider copying the required code. Large dependencies bloat the build classpath, increase memory usage, 
and pollute autocompletion in user's `build.gradle.kts` files. Similarly, avoid using unnecessary plugin dependencies,
for example adding `osdetector-gradle-plugin` when simply checking the OS type.
- **Handle unstable dependencies with care**: **Never** include dependencies that declare no runtime stability between
versions, for example Google's protobuf runtime or ANTLR4. The shared build classpath means conflicts between different
versions used by different plugins can lead to runtime crashes or unexpected behavior. This puts users in a difficult
position in fixing their build as the only option is to revert to older versions of plugins that have matching unstable
dependency. Ideally, use libraries with a stable runtime, for example [wire](https://github.com/square/wire) for protos.
If you must use an unstable library, you should **relocate** your unstable library under a different Java package and
bundle it inside your own plugin artifact using a tool such as [Shadow Gradle Plugin](https://gradleup.com/shadow/configuration/relocation/).
- **Avoid bundling un-relocated libraries**: **Never** bundle un-relocated libraries inside your own artifact. This leads
to duplicate classes on the classpath, causing debugging nightmares as shared in [Tony's post](https://dev.to/autonomousapps/this-is-why-we-cant-have-nice-things-when-pom-files-lie-3lm5).
- **Use dependency scopes correctly**: use `implementation` (preferred) and `api` scopes for your plugin dependencies.
Hopefully in the future Gradle will improve and remove implementation details from `build.gradle.kts` compile classpath
drastically speeding up compilation and cleaning up user's autocomplete.
- **Prefer `compileOnly` for plugin dependencies**: When your plugin depends on another plugin, use `compileOnly` scope
where possible. This prevents forcing a specific version of that plugin on your users.
- **Avoid default packages**: **Never** ship plugins classes without a package (default package). While it allows for
very nifty import-less usage in `build.gradle.kts`, it pollutes the global namespace.
- **Maintain compatibility**: Follow the principles outlined in [Conservative librarian post]({% post_url 2025-01-10-Conservative-Librarian %})
regarding Java and Kotlin version compatibility. Also, keep the required Gradle version reasonably low, to allow users
to upGradle on their own pace.

By adhering to these principles, you can create robust and maintainable Gradle plugins that minimize the risk of
dependency conflicts and improve the overall build experience for your users.
