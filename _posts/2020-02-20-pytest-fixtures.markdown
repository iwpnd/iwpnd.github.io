---
layout: post
title: Stop killing kittens. Or, how to use pytest.fixtures to remove redundancy in your test suite
tags: [python, pytest]
categories: python pytest
date: 2020-02-20 10:13:37 +0200
toc: false
---

Last week I wrote about how [pytest.mark.parametrizing](https://iwpnd.pw/articles/2020-02/pytest-parametrize) can be used to remove some redundancy in your test suite. Today let's talk `pytest.fixture` and how it helps you to clean up the mess that is your test suite with reusable variables, connections and/or objects.

<p align="center">
<img src="/img/2020-02-pytest-fixtures/duplication-kills-kittens.jpg" alt="duplication kills kittens">
</p>

## stop killing kittens with pytest.fixture

If you [obey the testing goat](https://www.obeythetestinggoat.com/) like you should, you practice Test-Driven-Development. Therefore you make sure to code in small incremental steps. During the refactoring phase, you will notice that repetition is omnipresent. You always pass the same data to the tests and you often instantiate the same objects. However, instead of running the same code for every test, you can attach so-called fixture functions to the test, that run and return the data to the test when needed in a reliable, consistent and repeatable manner.

Let's try this in an example.

```python
class MyClass:
    def __init__(self, name: str, foo: int, bar: int) -> None:
        self.name = name
        self.foo = foo
        self.bar = bar
```

So, you have a class `MyClass` and you always use the same instance of the class in your test suite.

```python
# test_myclass.py
import MyClass

def test_myclass_1():
    myclass = MyClass(name="Panda", foo=13, bar=37)
    
    assert myclass.foo + 24 == 37

def test_myclass_2():
    myclass = MyClass(name="Panda", foo=13, bar=37)
    
    assert myclass.bar - 24 == 13
```

Instead of instantiating MyClass on every test, you can add a `pytest.fixture` and pass it as an argument to every test. Fixture functions are registered by marking them with `@pytest.fixture`.

```python
# test_myclass.py
import MyClass
import pytest

@pytest.fixture
def myclass():
    myclass = MyClass(name="Panda", foo=13, bar=37)
    return myclass

def test_myclass_1(myclass):
    assert myclass.foo + 24 == 37

def test_myclass_2(myclass):
    assert myclass.bar - 24 == 13

```

Now pytest finds the `test_myclass_1` and `test_myclass_2` test functions, because of the `test_` prefix. The test functions need a function argument named `myclass`. A matching fixture function is discovered by looking for a fixture-marked function named `myclass`. Pytest calls `myclass()` to create an instance of `MyClass` and returns the instance to either test function.

Defining the fixture within `test_myclass.py` comes with the trade-off that you limit the scope of your fixture to the `test_myclass.py` test file - it cannot be used in another test file that way. So what do you do? Define the same fixture in another class? No, that would cause code repetition again. Luckily `pytest` has a solution for that.

## conftest

To make fixtures available to your entire test suite you use a `conftest.py` file in your tests folder. In this file you store the fixtures you plan to use across your tests.

```python
# conftest.py
import pytest

@pytest.fixture
def myclass():
    myclass = MyClass(name="Panda", foo=13, bar=37)
    return myclass
```

```
└── tests
    ├── conftest.py
    ├── integration
    │   └── test_integration.py
    └── unit
        ├── test_myclass_1.py
        └── test_myclass_2.py
```

Now that `conftest.py` is present. For every test function that has `myclass`, pytest will pass the return value of `myclass()` fixture to the test function.

## conclusion

Stop killing kittens. 

Instead leverage fixtures and the concept of dependency injection, which means that an object (the fixture) supplies the dependencies to another object (the test function). This concept makes for a very modular, better manageable and repetition-free test suites.