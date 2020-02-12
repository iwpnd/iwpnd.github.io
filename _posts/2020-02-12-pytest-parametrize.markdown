---
layout: post
title: use pytest.mark.parametrize to remove redundancy in your test suite
tags: [python, pytest]
categories: python pytest
date: 2020-02-12 10:13:37 +0200
toc: false
---

The further you go down the rabbit hole that is programming, the more you will get exposed to a thing called unit testing, or testing in general and eventually to test-driven development. There is a couple of things you need to get started. You need to understand  [what testing is, and why you need to test](https://jeffknupp.com/blog/2013/12/09/improve-your-python-understanding-unit-testing/) your code and you will have to face the hard reality:

<p align="center">
<img src="/img/2020-02-pytest/no.png" alt="just no">
</p>

After that, you will pick your test framework and get to work. I opted for [pytest](https://docs.pytest.org/en/latest/) for no particular reason at first. I noticed it was used in other teams at work, and I found it to be mentioned a little more often at conferences than other frameworks. I got the hang on it fairly quickly, yet I'm sure to not tap into its full potential and I'm still discovering things the further I get that make testing even more efficient. One of those things is the [parametrizing of test functions](http://doc.pytest.org/en/latest/parametrize.html#parametrize-basics).

## example function

Say you have a simple function that takes two `float` or `int` values and adds them together and throws a custom `TypeError` if the values your function received are not int or float.

```python
def add(a: Union[float, int], b: Union[float, int]) -> Union[float, int]:

    if any([isinstance(a, _type) for _type in [int, float]]) and any(
        [isinstance(b, _type) for _type in [int, float]]
    ):
        return a + b
    else:
        raise TypeError("a and b can either be int or float")
```

Now you want to test that function using pytest

```python
def test_example_success():
    result = add(a=2, b=3)
    assert result == 5
```
<br>
<p align="center">
<img src="/img/2020-02-pytest/screenshot_test.png" alt="pytest output simple test">
</p>
<br>

  
and it passes. No big surprise here.

### redundancy

So what you want to do now is to check whether `TypeError` is ever raised, if the input is something other than an `int` or `float`.
My naive understanding was, that I would just create a couple of test functions, that would test different types of inputs and see if `pytest.raises` would raise that `TypeError`.


```python
def test_example_add_fails_string():
    with pytest.raises(TypeError):
        result = add(a="2", b="3")


def test_example_add_fails_list():
    with pytest.raises(TypeError):
        result = add(a=[2], b=[3])
```
  
<br>
<p align="center">
<img src="/img/2020-02-pytest/screenshot_redundancy.png" alt="pytest output redundant tests">
</p>
<br>

While that works just fine, there is quite some redundancy in here, even if we are just testing that simple `add()` function. So to avoid that we can use the `pytest.mark.parametrize` decorator that allows us to test a multitude of arguments against the same test functions, without having to duplicate the test function itself.

### parametrizing

```python
@pytest.mark.parametrize(
    "a, b, expectation", # parameter for the test function
    [
        ("3", 4, pytest.raises(TypeError)),
        (3, "4", pytest.raises(TypeError)),
        (["3"], ["4"], pytest.raises(TypeError)),
        ("3", "4", pytest.raises(TypeError)), 
        # [...]
    ]
)
def test_example_add_raises(a, b, expectation):
    with expectation:
        result = add(a,b)
```

<br>
<p align="center">
<img src="/img/2020-02-pytest/screenshot_result_parametrize.png" alt="pytest output parametrized test">
</p>
<br>

As you can see pytest runs the same test function but separately with the parameters that you set in the `decorator` which essentially removes code duplication in your unit tests. And since every set of input parameters is appended in brackets in the pytest log, you can easily spot which input does not behave as intended. Of course, you can use the same idea to test the input that you want to succeed.


```python
@pytest.mark.parametrize(
    "a, b, expectation", 
    [
        (1, 3, 4),
        (1.3, 3.7, 5),
        (-1, 4, 3)
    ]
)
def test_example_success(a, b, expectation):
    result = add(a=a, b=b)
    assert result == expectation
```

<br>
<p align="center">
<img src="/img/2020-02-pytest/screenshot_parametrize-success.png" alt="pytest output parametrized test success">
</p>
<br>

That's it for a quick introduction. There's a lot more to learn on [parametrize-basics](http://doc.pytest.org/en/latest/parametrize.html#parametrize-basics) and even more when you went [past the basics](http://doc.pytest.org/en/latest/example/parametrize.html).

Next up, `fixtures`.
