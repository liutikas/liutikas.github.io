---
layout: post
title:  Relationship Status of minSdk, compileSdk, targetSdk
header: header-eng
---

It is year 2026 and yet Android developers continue to be mystified about `minSdk`, `compileSdk`, and `targetSdk`.
There are some myths and false claims floating in the community that I would like to clear up.

**minSdk** represents the oldest version of Android your application or library wants to support. For example, if you
only want to support Android Marshmallow (API 23) and newer, you set your `minSdk { version = release(23) }`. This
informs build tools such as R8 on what code optimizations are safe to make and Android Lint on what API level your code
will run on, so `NewApi` checks can work correctly. Generally `minSdk <= compileSdk`. `minSdk` has no relationship to
`targetSdk`. Bumping `minSdk` version does not generally force any tooling upgrades for library consumers, but they
might be forced to bump their application `minSdk` version.

**compileSdk** represents the version of Android APIs you want to be able to use in your application or library.
Android APIs are generally additive, but in some rare cases we remove APIs, for example `android.hardware.fingerprint.FingerprintManager`
that was removed in API 36. This additive property means that if you pick a `compileSdk { version = release(36) }` then
you have access to APIs added in 36 and all the previous versions of Android (except for removals). `compileSdk` used
for an application has no effect on the runtime behavior of this application, in other words if you build your app
with `compileSdk` 35 vs 36 your application will behave in the exact same way. `compileSdk` has no direct relationship
with `targetSdk` besides the fact that new APIs and new behaviors in the OS used to ship at the same cadence. However,
even this is no longer true, as minor SDK releases (e.g. 36.1) have no behavior changes and thus no matching `targetSdk`.
Bumping `compileSdk` version forces library consumers to upgrade their Android Gradle Plugin version ([compileSdk to minimum AGP version mapping](https://developer.android.com/build/releases/about-agp#api-level-support))
as new SDK releases frequently come with changes that are not compatible with old tools.

**targetSdk** represents the version the Android runtime behavior of your application. As Android evolves, we try very
hard to avoid breaking existing applications already in the Play Store and as a result we sometimes gate runtime behavior
changes behind an explicit `opt-in` by the application and this mechanism is `targetSdk`. When your application goes from
`targetSdk { version = release(35) }` to `targetSdk { version = release(36) }` you are telling the OS that you acknowledge
that your app will now get [the new behaviors](https://developer.android.com/about/versions/16/behavior-changes-16).
As noted above, there is no relationship with between `targetSdk` and `compileSdk` or `targetSdk` and `minSdk`, namely
your `targetSdk` can be a higher number than `compileSdk` and that is a completely reasonable state if you are happy
with the new runtime behaviors and no access to the APIs. `targetSdk` has no meaning for a library, because only the
final value set by the application has impact on the behavior. However, in a library case, you want to set `lint.targetSdk`
and `testOptions.targetSdk` to the latest API version to make sure that your library behaves correctly when the consumers
pick that version in their application.

## Why might you change one of these values?

**minSdk** - either your team no longer cares about supporting old Android versions or a library you depend on no longer
does and forces you to change your `minSdk` version. For example, AndroidX libraries moved from `minSdk` 21 to 23 in 2025.

**compileSdk** - either your team wants access to new shiny Android APIs or a library you depend on does and forces you
to change your `compileSdk` version. For example, `androidx.compose` adopted `compileSdk` 35 in 2025.

**targetSdk** - either your team wants a new runtime behavior or Google Play Store is no longer allowing you to publish
an application targeting that `targetSdk`. Libraries have *no* effect on your `targetSdk` version.
