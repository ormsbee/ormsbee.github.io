---
layout: post
title:  "Empathy and Error Logging"
date:   2015-07-21 21:55:57
# categories: logging
---

There are [good guidelines](http://dev.splunk.com/view/logging-best-practices/SP-CAAADP6)
for error logging out there. They tell you to use timestamps, UUIDs, levels,
categories, source line numbers, etc. All of these are great to know, but the
most important part is thinking hard about who your audience is, and being able
to imagine yourself in their shoes.


When you're writing an error log message, you're not writing it for yourself.
You're writing it for the poor sap who has to fix things when they break on a
Saturday afternoon, four years after you wrote it and two years after you left
the company. That log message is your saving throw into the future, with the
potential to save someone hours of work.

Keep in mind that if someone is staring at an error message, something has quite
likely escaped the QA process. Assume that you are having a one way conversation
with a person who was woken up by a pager at 2AM, is under intense pressure to
fix things, and is completely unfamiliar with that part of the codebase. What
that person desperately needs is context. Your error log message should convey:

1. What failed?
2. Why did it fail?
3. What was the system trying to do at the time?
4. How can the problem be reproduced?
5. What specific assumptions were violated?

Some examples based on things I've seen in various systems:

```python
# Utterly useless
logging.error("This should never happen.")
```

It is aggravating beyond belief to see tens of thousands of copies of this in
the logs during a crisis. Never do this, no matter how impossible the situation
might seem. At the very least, explain why you think it should never happen.

```python
# Slightly better
logging.error("Failed to create enrollment.")
```

It's really easy to write code like this without thinking about it. Some
condition fails, you catch an exception, and then you log. But there's no
context here, and it's difficult to know where to start debugging the problem.

```python
# Reproducible
logging.error("Failed to enroll user %s in course %s", user.id, course.id)
```

Ok, now we know what user and course it failed with. So we can just go to the
production database, extract information about that user and course, copy that
information to our local dev environment, and figure out what things affect
enrollment eligibility...

```python
# Helpful context
logging.error(
    "Failed to enroll user %s in course %s: course is closed (enrollment_start=%s, enrollment_end=%s)",
    user.id, course.id, course.enrollment_start, course.enrollment_end
)
```

As the author, that might feel silly. Maybe this code is only ever reached if
the enrollment was advertised, and if it wasn't in the active enrollment period,
it wouldn't be advertised.

How does something weird like this sneak in? Say for some reason you're storing
the enrollment start and end dates as ISO 8601 date strings. It's such a simple
check that someone didn't really think much of some code duplication, or didn't
properly refactor old code. Then someone decided to change it so that
`course.enrollment_end` is no longer mandatory, and used `None` to indicate that
there is no closing date for enrollment.

```python
>>> # Check: course.enrollment_start <= today <= course.enrollment_end ?
>>> "2015-07-18" <= "2015-07-21" <= None
False
```

Who knew Python would let us compare strings and None?

Clearly there should have been tests for this, and the gating logic should have
been more centralized. But sometimes stuff like this happens, and the error
message is your last opportunity to do the right thing by your colleagues.
