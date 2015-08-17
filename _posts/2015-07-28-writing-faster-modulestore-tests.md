---
layout: post
title:  Writing Faster ModuleStore Tests
date:   2015-07-28 05:20:39
# categories: "tests"
---

Most developers who have added features to edx-platform are familiar with
`ModuleStoreTestCase`. If your tests exercise anything relating to courseware
content (even it's just creating an empty course), inheriting from this class
will ensure that data gets cleaned up properly between individual tests. This is
extremely valuable, but can also be wasteful in many situations. During last
week's hackathon, I created [a faster alternative](https://github.com/edx/edx-platform/pull/9070)
called `SharedModuleStoreTestCase`.


Unlike `ModuleStoreTestCase`, `SharedModuleStoreTestCase` only does
`ModuleStore` cleanup at the `tearDownClass()` level. It's meant to be employed
in situations where one or a small handful of courses can be initialized up
front, and then shared in a read-only manner across many tests. This usage
pattern is commonly found in LMS tests, which often simply recreate the same
course over and over again in their `setUp()` methods.

# Performance Impact

To get an idea of the effect it could have, I switched over a few test modules
as part of my hackathon work. These are only rough figures, as they are based on
a relatively small number of Jenkins test runs. That being said, the results are
promising:

| File                                              | # Tests | Before  | After | Delta |
| :------------------------------------------------ | -------:| -------:| -----:| -----:|
| lms/djangoapps/ccx/tests/test_ccx_modulestore.py  |       5 |     38s |    4s |  -89% |
| lms/djangoapps/discussion_api/tests/test_api.py   |     409 |  2m 45s |   51s |  -69% |
| lms/djangoapps/teams/tests/test_views.py          |     152 |  1m 17s |   33s |  -57% |

So how do you convert your own tests?

## Making the Switch

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

If you are modifying `self.course` in your individual test functions, then
this is perfect, and you should continue to use `ModuleStoreTestCase`. However,
if you're just setting up the course once and treating it as read-only in your
tests, you can now do this instead:

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

It's important that Django ORM operations remain in `setUp()`. Any models that
you create in `setUpClass()` must be manually deleted in your `tearDownClass()`
method — `SharedModuleStoreTestCase` will not properly clean them up. Even if
you're careful, you're still likely to break other tests in the system in
unpredictable ways because they make bad assumptions about sequences and what
IDs will be created when they set up their data. This can be extremely tedious
to debug.

When we upgrade to Django 1.8, you'll be able to use
[`setUpTestData()`](https://docs.djangoproject.com/en/1.8/topics/testing/tools/#django.test.TestCase.setUpTestData)
to safely do class-level initialization of Django models with automatic cleanup.
Please wait for that upgrade and place model manipulations in `setUp()` for now,
even if it is a bit slower.

## Which Tests Should I Convert?

The easiest place to hunt for test optimization targets is the Jenkins 
[test build report](https://build.testeng.edx.org/job/edx-platform-python-unittests-master/lastStableBuild/testReport/).
Click on "Duration" to sort by that column.

![Jenkins Test Report](/assets/writing-faster-modulestore-tests/which_tests.png)

We primarily want to target expensive tests that either create complex course
data (e.g. CCX) or have simple course data but many, many tests (e.g.
discussions). Creating even the simplest course takes about 250-300ms or so,
which really adds up when using tools like [`ddt`](http://ddt.readthedocs.org)
that effectively multiply the number of tests in a class.

## Overall Takeaway

Test data creation and cleanup can be an expensive, and the `ModuleStore` is a
prime example of that. I hope that `SharedModuleStoreTestCase` can be a useful
tool for bringing down test execution times. But beyond that, I hope that
understanding why it works will allow us to design faster test suites in general.
