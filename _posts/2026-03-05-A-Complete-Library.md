---
layout: post
title:  A Complete Library
header: header-eng
mastodon: 116178740307843639
---

Block 40% layoffs has spurred a lot of discussions about open source projects attached to a for-profit corporation.
In many cases such projects are largely hero-efforts from a few amazing individual employees rather than the
corporation at large. This in turn can cause folks to try to assess this situation in terms of impact to their own
project and induce panic to move away from these OSS projects. With this context, I want to discuss the categories of
the these OSS projects and compare risk levels for using each category.

In androidx we categorize our libraries into the following buckets (conditions applied in order):
- **Abandoned Experimental**: never released stable and has shipped more than 3 months ago
- **Experimental**: never released stable
- **Complete**: latest version is a stable version
- **Actively developed**: latest version release less than 3 months ago
- **Abandoned Stable**: everything else

To sort these categories by risk starting with least risky, we have:
- **Complete**
- **Actively developed**
- **Experimental**
- **Abandoned Stable**
- **Abandoned Experimental**

Complete libraries are effectively done, they are battle-tested, and they serve the goal that was initially scoped,
folks in UK would say "Does exactly what it says on the tin". `androidx.palette` is an example of such a library. If
it has the APIs to do what you want, it's perfect, and it comes with a low risk for your own project. One could argue
that `okio` and `okhttp` are largely in the same category. It is ok and sometimes even good for such libraries to stop
the development.

Abandoned libraries on the other hand are on the opposite side of the risk scale, they come with a very large potential
cost to your project, they are effectively unfinished and unsupported.

Importantly, there are other factors besides the version/release time that affect the category. For example a deep
reliance on a fast moving dependency such as a stable library that relies on a Kotlin compiler plugin. This library
might not be a complete library due to the fact that it can never stop the development without breaking, thus adding
a high risk factor to your project.

It is important to assess your project risks. However, make sure consider that a library that you rely on might still be
low risk to you even if it looses all corporate contributors.
