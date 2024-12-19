---
layout: post
title:  What The Distribution - What Is Inside Gradle Distribution Zips?
header: header-eng
---

When I saw [this post](https://bsky.app/profile/goetas.com/post/3ldlafyd62c2s)

![A BlueSky post asking why is Gradle ~200MB in size](/assets/2024-12-18-bluesky.png){:width="600px"}

I first giggled, but then honestly wanted to answer their question. How here it goes!

Gradle has two flavors of distributions: `gradle-8.11.1-all.zip` and `gradle-8.11.1-bin.zip`, their
respective compressed sizes are 219.4MB and 130MB, whereas uncompressed sizes are 455.3MB and 144.5MB.
The difference between the two is `docs` (HTML/CSS/PDF/images for offline documentation) and `src` (all of Gradle sources),
making it easy to tinker with Gradle entirely offline. Given that Gradle is kind of useless without the internet, stick
to [Gradle docs website](https://docs.gradle.org/) and always use `-bin` version, your devs and CI will thank you!

Now, if we dive into `gradle-8.11.1-bin.zip`, what's inside of that?
![A screenshot showing libs, bin, init.d directories with their respective sizes](/assets/2024-12-18-gradle-bin.png)

- `bin` contains a bash and Window batch file scripts to start Gradle
- `init.d` is a directory that allows custom Gradle distributions to add `.gradle` init scripts and each one gets 
executed at the start of the build.
- `lib` is what takes pretty much the entire space with 311 `.jar` files.

Let's dig into what we have inside of `lib`

![A screenshot showing a list of files with their sizes inside of libs/ directory inside the zip](/assets/2024-12-18-libs.png)

From here, we see that Kotlin compiler is 58.3MB (40% of the distribution), groovy compiler adds 8.1MB (5.6%).
There are 166 `gradle-*.jar` files taking up 24.8MB (17%).

That is a lot of space, and Gradle has been growing - `gradle-8.0-bin.zip` was 118.4MB, so 10% growth since
February 2023.

A sad side note, the Kotlin compiler that ships inside Gradle can only be used to compile `.gradle.kts` files, so
if you try to build real Kotlin, you need to fetch a separate new copy through Kotlin Gradle Plugin!
