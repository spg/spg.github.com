---
layout: post
title: Short introduction to pytest
---

When tasked with setting up a testing environment for our team at Poka, I knew I wanted the testing experience to be:

* simple
* fun

Since writing tests might not be viewed by all as an exciting task (though I love writing tests), I wanted to make the whole process as effortless as possible.

I initially tried the `unittest` library that comes bundled with Python.
One of the pains I found with `unittest` is the amount of boilerplate code required to write a test. For example:

```python
import unittest

def can_explode(item):
    return item in ["Note 7", "hoverboard"]

class TestExplode(unittest.TestCase):
    def test_can_explode(self):
        self.assertTrue(can_explode("Note 7"))

if __name__ == '__main__':
    unittest.main()
```

In the example above, in order to write the simplest test case, I had to: import the `unittest` module, create a test class, write the `main` sentinel, include a `self` parameter in my test function and use the bundled `assertTrue` function.

Compare this with the equivalent using `pytest`:

```python
def can_explode(item):
    return item in ["Note 7", "hoverboard"]

def test_can_explode():
    assert can_explode("Note 7")
```

Notice how minimal this test is compared to the first `unittest` sample. As you can see, the `pytest` example requires a lot less code. 
The act of writing a test has now been reduced to its simplest form: create a function that begins with `test_*` and use the `assert` keyword to express your assertions.

Speaking of assertions, one other nice feature of `pytest` is that you can use Python's `assert` keyword for declaring your assertions. This is once again much simpler than having to call `self.assertTrue(expr)`, as it does not require using external functions.
Also, `pytest` does a great job of displaying the diff when yor assertion fails, for example:
```shell
====================== FAILURES ======================
___________________ test_exploded ____________________

    def test_exploded():
>       assert "exploded" == "exp1oded"
E       assert 'exploded' == 'exp1oded'
E         - exploded
E         ?    ^
E         + exp1oded
E         ?    ^

```
shows you the exact spot where your strings differ.

Another facet of making tests accessible is to allow your team to run them easily. `pytest`'s test discovery algorithm makes it really simple to get your tests picked up.
To write a test, you simply create a Python module that matches `test_*.py` (e.g. `test_models.py`) and pytest will collect all functions whose names begin with  `test_`.
You can organize your tests folder hierarchy as you please, `pytest` will happily walk through your directories to find the `test_*` functions inside the `test_*.py` modules.
Simply fire up `py.test` from the command line in your current directory and all your tests will be collected and executed:

```shell
$ py.test
======================= test session starts ===========================
platform darwin -- Python 2.7.9, pytest-2.8.4, py-1.4.31, pluggy-0.3.1
rootdir: /Users/spg/myproject
collected 5 items

tests/test_models.py ...
tests/utils/test_tools.py ..

===================== 5 passed in 0.13 seconds ========================
```

## Some cool features

### fixtures
One of the most interesting features of `pytest` is its fixtures system. 
Basically, fixtures are a replacement for the setup and teardown process we find in many other test frameworks.

```python
import pytest
import explodingobjects


def good_christmas_gift(item):
    conn = explodingobjects.connect()
    return not conn.exists(item)


@pytest.fixture
def explodingobjects_connection(request):
    conn = explodingobjects.connect()
    conn.add(["Note 7", "hoverboard"])
    def fin():
        conn.clear()

    request.addfinalizer(fin)


def test_good_christmas_gift(explodingobjects_connection):
    assert not good_christmas_gift("Note 7")
```
Here we declared a `pytest` fixture using the `@pytest.fixture` decorator, and we injected this fixture as a parameter to our `test_good_christmas_gift` function. 
Whenever my `test_good_christmas_gift` is executed, pytest will execute `explodingobjects_connection` prior to executing my test. In our case,
we're creating a database for testing. When the test is over, the fixture's finalizer (here, the `fin` function) will be executed, in our case clearing the test database. 
Note that the finalizer will be executed whether the test has passed or failed.

If you want to automatically use a fixture for any tests in our module, use the `autouse=True` parameter, like this:

```python
import pytest
import explodingobjects


def good_christmas_gift(item):
    conn = explodingobjects.connect()
    return not conn.exists(item)


@pytest.fixture(autouse=True)
def explodingobjects_connection(request):
    conn = explodingobjects.connect()
    conn.add(["Note 7", "hoverboard"])
    def fin():
        conn.clear()

    request.addfinalizer(fin)


def test_good_christmas_gift1():
    assert not good_christmas_gift("Note 7")

def test_good_christmas_gift2():
    assert not good_christmas_gift("hoverboard")
```
Now, we don't have to inject `explodingobjects_connection` into our test functions.
The database will be populated before each test, and cleared after each test.

What if I want to retrieve the database connection from within my test? Look at this:

```python
import pytest
import explodingobjects


def good_christmas_gift(item):
    conn = explodingobjects.connect()
    return not conn.exists(item)


@pytest.fixture
def explodingobjects_connection(request):
    conn = explodingobjects.connect()
    conn.add(["Note 7", "hoverboard"])
    def fin():
        conn.clear()

    request.addfinalizer(fin)

    return conn


def test_good_christmas_giftexplodingobjects_connection):
    explodingobjects_connection.add("Note 8")
    assert not good_christmas_gift("Note 8")
```
The injected parameter `explodingobjects_connection` will be the actual value returned by the `explodingobjects_connection` function. 
I can now use my database connection from within my test.

### parametrize
Another small but useful feature of pytest is the `parametrize` marker.
It will generate tests based on a set of test cases.
This feature allows us to transform this test:
```python
def can_explode(item):
    return item in ["Note 7", "hoverboard"]


def test_can_explode():
    assert can_explode("Note 7")
    assert not can_explode("Furby")
    assert can_explode("hoverboard")
```
into 
```python
import pytest


def can_explode(item):
    return item in ["Note 7", "hoverboard"]


@pytest.mark.parametrize("item, could_explode", [
    ("Note 7", True),
    ("Furby", False),
    ("hoverboard", True)
])
def test_can_explode(item, could_explode):
    assert can_explode(item) == could_explode
```
The first parameter to the `parametrize` annotation is the name of the parameters (`"item, could_explode"`) to be injected in the test function. The second parameter is a list of tuples, where each tuple will be injected as parameters to our test function.

When running this test, `pytest` will now create 3 individual tests instead of one, and execute those 3 tests independently.
In the first example, if the first test case (`assert can_explode("Note 7")`) were to fail, then the next 2 test cases would not be executed.
This can be problematic if the 2nd or 3rd test cases are also failing, as you won't know they are failing until you have fixed the 1st test case.


## Third party plugins

### parallel testing
You might want to run your tests in parallel in order to make your whole test suite run faster.
Thankfully, there's a plugin called `pytest-xdist` that does just that. Install pytest-xdist, and now:

```shell
py.test -n 8 
```
will run your tests using 8 CPUs. 

### random testing 
A property of good test suites is the ability to run all tests in any order. This will help you in finding tests that
have dependencies on other tests (such as relying on the output of a previous test in order to work).
You can use `pytest-random` in order to randomize your test execution:

```shell
py.test --random
```

and if you also want to run your tests in parallel:

```shell
py.test -n 8 --random 
```


## Wrap up
`pytest` has the advantage of simplicity over the bundled package `unittest`. Tests are shorter to write and its assertions are more expressive.
You can easily declare setup and teardown steps using fixtures. Repeating test logic over many test cases is made easy using
parametrize. Finally, `pytest` has many plugins that make allow powerful additions to your test suite such as parallel test execution and randomization. 
