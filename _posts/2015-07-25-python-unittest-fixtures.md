---
layout: post
title: "Python unittest Fixtures"
modified: 2015-07-25 15:05:56 -0400
category: posts
tags: [python, unittest, fixtures, testing, unit testing, test case]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

The Python 2.7 `unittest` package (`unittest2` in Python < 2.7) provides several mechanisms for defining test fixtures of increasing granularity at the module, class, and method levels. This post gives some example code for these fixtures and shows in what order they execute.

## Test fixtures

Test fixtures provide the necessary initial conditions and configure a fixed state for software tests. They also clean up after testing completes. Test fixtures provide a stable baseline for consistent, repeatable testing and allow for test initialization and clean up code to remain separate from the tests themselves.

Sometimes the same test fixture can apply to several tests. For example, each test in a test suite used to exercise an interface to a database may need to configure (or mock) a database connection. It can be time consuming or resource intensive to bring up and tear down this connection for each test. Thankfully, the `unittest` framework provides three scopes for test fixtures; fixtures can be defined at the individual test, class or module level.

### Module-scope fixtures

Module-scope test fixtures provide initial conditions for all tests within that module and clean up afterward using the `setUpModule` and `tearDownModule` functions. Any set up needed by all tests in a module should live in these two functions. When running a test suite, the `unittest` package executes `setUpModule` before executing any tests in that module and then executes `tearDownModule` after all tests in that module have completed.

### Class-scope fixtures

Class-scope test fixtures provide the common initial conditions for all tests located within the same class. Setup is done in the `setUpClass` class method and clean up is done in the `tearDownClass` method. Since these are class methods, they both require the `@classmethod` decorator. Each method is only run once for all tests in that class; `setUpClass` is run before all tests in the class and `tearDownClass` is ran after all tests in the class.

### Test-scope fixtures

Test-scope test fixtures provide setup and tear down for each test in the same class. The `setUp` instance method handles setup and the `tearDown` instance method handles any necessary cleanup. These methods run before and after each test, compared to class-scope fixtures that are run once for all tests in the same class.

## Example test case

Take Python files in this example are arranged like so:

{% highlight bash %}
test_examples/
├── __init__.py
├── package1
│   ├── __init__.py
│   └── subpackage1
│       ├── __init__.py
│       ├── module1.py
│       └── module2.py
├── package2
    ├── __init__.py
    ├── subpackage1
    │   ├── __init__.py
    │   ├── module1.py
    │   └── module2.py
    └── subpackage2
        ├── __init__.py
        └── module1.py
{% endhighlight %}

Each `module.py` file contains the following example test code:

{% highlight python %}
from unittest import TestCase

def setUpModule():
    print "setUpModule: " + __name__ + " set up"
    print

def tearDownModule():
    print "setUpModule: " + __name__ + " tear down"
    print


class TestCase1(TestCase):

    @classmethod
    def setUpClass(cls):
        print "    setUpClass: " + cls.__name__ + " set up"
        print

    @classmethod
    def tearDownClass(cls):
        print "    tearDownClass: " + cls.__name__ + " tear down"
        print

    def setUp(self):
        print "        setUp: " + self.id() + " set up"
        print

    def tearDown(self):
        print "        tearDown: " + self.id() + " tear down"
        print

    def test_one(self):
        print "            " + self.id() + " running"
        print

    def test_two(self):
        print "            " + self.id() + " running"
        print

class TestCase@(TestCase):

    @classmethod
    def setUpClass(cls):
        print "    setUpClass: " + cls.__name__ + " set up"
        print

    @classmethod
    def tearDownClass(cls):
        print "    tearDownClass: " + cls.__name__ + " tear down"
        print

    def setUp(self):
        print "        setUp: " + self.id() + " set up"
        print

    def tearDown(self):
        print "        tearDown: " + self.id() + " tear down"
        print

    def test_one(self):
        print "            " + self.id() + " running"
        print

    def test_two(self):
        print "            " + self.id() + " running"
        print
{% endhighlight %}

This example code consists of a `setUpModule` function, a `tearDownModule` function, and an example `TestCase` class. The class contains setup and tear down class methods that execute once as well as regular set up and tear down methods that execute before and after each test. Each function prints out descriptive information to standard out to make it easier to see the order in which the code is run.

Enter `python -m unittest . "*.py"` in the terminal to run the tests, resulting in the following output:

{% highlight bash %}
setUpModule: test_examples.package1.subpackage1.module1 set up
    setUpClass: TestCase1 set up
        setUp: test_examples.package1.subpackage1.module1.TestCase1.test_one set up
            test_examples.package1.subpackage1.module1.TestCase1.test_one running
        tearDown: test_examples.package1.subpackage1.module1.TestCase1.test_one tear down
        setUp: test_examples.package1.subpackage1.module1.TestCase1.test_two set up
            test_examples.package1.subpackage1.module1.TestCase1.test_two running
        tearDown: test_examples.package1.subpackage1.module1.TestCase1.test_two tear down
    tearDownClass: TestCase1 tear down
    setUpClass: TestCase2 set up
        setUp: test_examples.package1.subpackage1.module1.TestCase2.test_one set up
            test_examples.package1.subpackage1.module1.TestCase2.test_one running
        tearDown: test_examples.package1.subpackage1.module1.TestCase2.test_one tear down
        setUp: test_examples.package1.subpackage1.module1.TestCase2.test_two set up
            test_examples.package1.subpackage1.module1.TestCase2.test_two running
        tearDown: test_examples.package1.subpackage1.module1.TestCase2.test_two tear down
    tearDownClass: TestCase2 tear down
setUpModule: test_examples.package1.subpackage1.module1 tear down
setUpModule: test_examples.package1.subpackage1.module2 set up
    setUpClass: TestCase1 set up
        setUp: test_examples.package1.subpackage1.module2.TestCase1.test_one set up
            test_examples.package1.subpackage1.module2.TestCase1.test_one running
        tearDown: test_examples.package1.subpackage1.module2.TestCase1.test_one tear down
        setUp: test_examples.package1.subpackage1.module2.TestCase1.test_two set up
            test_examples.package1.subpackage1.module2.TestCase1.test_two running
        tearDown: test_examples.package1.subpackage1.module2.TestCase1.test_two tear down
    tearDownClass: TestCase1 tear down
    setUpClass: TestCase2 set up
        setUp: test_examples.package1.subpackage1.module2.TestCase2.test_one set up
            test_examples.package1.subpackage1.module2.TestCase2.test_one running
        tearDown: test_examples.package1.subpackage1.module2.TestCase2.test_one tear down
        setUp: test_examples.package1.subpackage1.module2.TestCase2.test_two set up
            test_examples.package1.subpackage1.module2.TestCase2.test_two running
        tearDown: test_examples.package1.subpackage1.module2.TestCase2.test_two tear down
    tearDownClass: TestCase2 tear down
tearDownModule: test_examples.package1.subpackage1.module2 tear down
setUpModule: test_examples.package2.subpackage1.module1 set up
    setUpClass: TestCase1 set up
        setUp: test_examples.package2.subpackage1.module1.TestCase1.test_one set up
            test_examples.package2.subpackage1.module1.TestCase1.test_one running
        tearDown: test_examples.package2.subpackage1.module1.TestCase1.test_one tear down
        setUp: test_examples.package2.subpackage1.module1.TestCase1.test_two set up
            test_examples.package2.subpackage1.module1.TestCase1.test_two running
        tearDown: test_examples.package2.subpackage1.module1.TestCase1.test_two tear down
    tearDownClass: TestCase1 tear down
    setUpClass: TestCase2 set up
        setUp: test_examples.package2.subpackage1.module1.TestCase2.test_one set up
            test_examples.package2.subpackage1.module1.TestCase2.test_one running
        tearDown: test_examples.package2.subpackage1.module1.TestCase2.test_one tear down
        setUp: test_examples.package2.subpackage1.module1.TestCase2.test_two set up
            test_examples.package2.subpackage1.module1.TestCase2.test_two running
        tearDown: test_examples.package2.subpackage1.module1.TestCase2.test_two tear down
    tearDownClass: TestCase2 tear down
tearDownModule: test_examples.package2.subpackage1.module1 tear down
setUpModule: test_examples.package2.subpackage1.module2 set up
    setUpClass: TestCase1 set up
        setUp: test_examples.package2.subpackage1.module2.TestCase1.test_one set up
            test_examples.package2.subpackage1.module2.TestCase1.test_one running
        tearDown: test_examples.package2.subpackage1.module2.TestCase1.test_one tear down
        setUp: test_examples.package2.subpackage1.module2.TestCase1.test_two set up
            test_examples.package2.subpackage1.module2.TestCase1.test_two running
        tearDown: test_examples.package2.subpackage1.module2.TestCase1.test_two tear down
    tearDownClass: TestCase1 tear down
    setUpClass: TestCase2 set up
        setUp: test_examples.package2.subpackage1.module2.TestCase2.test_one set up
            test_examples.package2.subpackage1.module2.TestCase2.test_one running
        tearDown: test_examples.package2.subpackage1.module2.TestCase2.test_one tear down
        setUp: test_examples.package2.subpackage1.module2.TestCase2.test_two set up
            test_examples.package2.subpackage1.module2.TestCase2.test_two running
        tearDown: test_examples.package2.subpackage1.module2.TestCase2.test_two tear down
    tearDownClass: TestCase2 tear down
tearDownModule: test_examples.package2.subpackage1.module2 tear down

----------------------------------------------------------------------
Ran 16 tests in 0.001s

OK
{% endhighlight %}

The indentation in the test results make the order of code execution more obvious.
