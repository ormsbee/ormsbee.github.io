---
layout: post
title:  Writing Faster ModuleStore Tests
date:   2015-07-28 05:20:39
categories: tests, performance
---

Most developers who have added features to edx-platform are familiar with
`ModuleStoreTestCase`. If your tests exercise anything relating to courseware
content (even it's just creating an empty course), inheriting from this class
will ensure that data gets cleaned up properly between individual tests. This is
extremely valuable, but it's also computationally expensive, and it turns out
that it's overkill for many of our test cases. To help speed things up, I
[recently added](https://github.com/edx/edx-platform/pull/9070)
`SharedModuleStoreTestCase` as part of a hackathon project.


Unlike `ModuleStoreTestCase`, `SharedModuleStoreTestCase` only does
`ModuleStore` cleanup at the class level. It's meant to be employed in
situations where one or a small handful of courses can be initialized once, and
then used in a read-only manner by many tests. This is a particularly common
practice for LMS tests, which often create the same course over and over again
in their `setUp()` methods.

I converted a few test modules to measure the effects. Keep in mind that these
numbers were using Jenkins reports across a very limited set of runs, and so are
extremely rough.

| File                                              | # Tests | Before  | After | Delta |
| :------------------------------------------------ | -------:| -------:| -----:| -----:|
| lms/djangoapps/ccx/tests/test_ccx_modulestore.py  |       5 |     38s |    4s |  -89% |
| lms/djangoapps/discussion_api/tests/test_api.py   |     409 |  2m 45s |   51s |  -69% |
| lms/djangoapps/teams/tests/test_views.py          |     152 |  1m 17s |   33s |  -57% |

Because course creation is relatively slow (~100-200ms), eliminating this per-test
overhead is often most noticeable on test cases that use 
[`ddt`](http://ddt.readthedocs.org) to generate many test function invocations.
That being said, the largest percentage gain came in `test_ccx_modulestore.py`,
which has only five tests. This is partially because the course it created was
considerably more complex than most, but I can't completely explain a 9.5x
speedup across five tests at this time.

So how do you convert your own tests?

## Make Your ModuleStore Tests Faster With This One Weird Trick

Most classes that inherit from `ModuleStoreTestCase` start something like this:

```python
from student.tests.factories import CourseEnrollmentFactory, UserFactory
from xmodule.modulestore.tests.django_utils import ModuleStoreTestCase
from xmodule.modulestore.tests.factories import CourseFactory, ItemFactory

class MyContentModifyingTestCase(ModuleStoreTestCase):

    def setUp(self):
        super(MyContentModifyingTestCase, self).setUp()
        self.course = CourseFactory.create()
        self.user = UserFactory.create()
        CourseEnrollmentFactory.create(user=self.user, course_id=self.course.id)
```

If you are modifying `self.course` in your tests, this the right way to go.
However, if you're just setting up the course once and treating the course as
read-only in your tests, you can now do this instead:

```python
from student.tests.factories import CourseEnrollmentFactory, UserFactory
from xmodule.modulestore.tests.django_utils import SharedModuleStoreTestCase
from xmodule.modulestore.tests.factories import CourseFactory, ItemFactory

class MySharedModuleStoreTestCase(SharedModuleStoreTestCase):
    @classmethod
    def setUpClass(cls):
        """Any ModuleStore course/content operations can go here."""
        super(MySharedModuleStoreTestCase, cls).setUpClass()
        cls.course = CourseFactory.create()        

    def setUp(self):
        """Django ORM operations still need to go in setUp() for now."""
        super(MySharedModuleStoreTestCase, self).setUp()
        self.user = UserFactory.create()
        CourseEnrollmentFactory.create(user=self.user, course_id=self.course.id)
```

An important note here is that Django ORM operations should still be put in
`setUp()`. Any models that you create in `setUpClass()` must be manually
deleted in your `tearDownClass()` method — `SharedModuleStoreTestCase` will not
properly clean them up. Even if you're careful, you're likely to break other
tests in the system in unpredictable ways because they make bad assumptions
about sequences and what IDs will be created when they set up their data. This
can be extremely tedious to debug.

When we upgrade to Django 1.8, you'll be able to use
[`setUpTestData()`](https://docs.djangoproject.com/en/1.8/topics/testing/tools/#django.test.TestCase.setUpTestData)
to safely do class-level initialization of Django models with automatic cleanup.
Please wait for that upgrade and place model manipulations in `setUp()` for now,
even if it is a bit slower.
