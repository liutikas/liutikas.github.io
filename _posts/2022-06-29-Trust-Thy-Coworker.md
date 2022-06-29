---
layout: post
title:  Trust Thy Coworker
header: header-eng
---

As software engineers, we often want to make sure that the code being added to
the project is of high quality, which is often achieved by having code review
process. Taking a small detour from Gradle, I wanted to share a few changes in
the code review process that my team adopted that led us to some positive outcomes.

One way to help review is with automated linters
([ktlint](https://github.com/pinterest/ktlint), [Checkstyle](https://checkstyle.sourceforge.io/),
[Android Lint](https://developer.android.com/studio/write/lint), and others).
These tools can help get nitpicks out of the way, like `please keep this list alpha-sorted`
or `when statement is preferred here instead of chained if`. When using them,
by the time the code is up for review, you are focusing on the fundamentals of
the change instead of important, but automatically enforceable issues. This is
especially critical when your team is distributed across many time-zones
as you might be getting only a single review cycle in per day.

The second and in my mind even more significant change was to pre-emptively
start giving approvals to land when the change is submit-worthy besides minor
shortcomings. This means trusting the author to do those fixes you suggested
and landing the change without another round of reviews. This can be a difficult
organizational change as it requires a lot of trust within the team, however
when it is implemented it enables higher velocity and makes change authors
feel more empowered while making reviewers save time in the long run.

Finally, a lot of it is about expectations, so writing down the basic rules
for your team's code review can help clarify what does `-2` on your change
actually means. [AndroidX Code Review Etiquette](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:code-review.md)
was something we wrote together as a team to keep everyone on the same page and
in the process dispelled some myths and established pre-emptive approval mentioned
above.
