---
layout: post
title:  Attention Pyramid
header: header-eng
mastodon: 116910269906404031
---

If you worked enough time with software, you must have crossed paths with the idea of a testing pyramid. It is a handy
aid to make sure tests are in the right places and quantity. It is especially helpful when a project is moving quick,
and you have a limited amount of time to spend on tests. It takes away some of the mental burden and team friction.

I think of the testing pyramid as an attention pyramid. It can be extended beyond tests to your production code. Where
should you spend your limited attention to get the best outcomes?

![Attention pyramid](/assets/2026-07-12-pyramid.png)

The lower levels of the pyramid tend to have a much higher impact, as a single bad decision can be multiplied by several
orders of magnitude more than something at a higher levels. Therefore, if you were going to cut some corners, it is best
to do that at top. In today's world of LLMs onslaught, our attention is even further strained.

I recommend using this attention pyramid lens when deciding how much time to spend authoring or reviewing a PR in your
codebase. For some changes, you might be better off reading and comprehending every single line of code, and for others
you can likely skip that with minimal consequences. It can also be impactful to build a consensus within your own team
as to which parts of your codebase belong to which level of attention. This allows the authors and reviewers to be
aligned on the expected amount of effort avoiding frustration on both sides.

Let's spend our attention wisely!
