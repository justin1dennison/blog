---
title: "Abuse Python for Fun(ction)s"
date: 2020-04-26T22:01:23-04:00
draft: false 
tags:
  - python
  - function composition
  - magic methods
  - operator overloading
  - not meant for production
  - decorators
  - python data model
---

I know that I shouldn't have these "crazy" ideas, but sometimes they just jump into my mind. Recently, I was speaking to a coworker that is on a Linux learning journey, and he asked me about some of the `bash` scripting nuances. We were discussing some of the different operations available in `bash` such as `@` and `|`. At that moment, I started thinking about how the `|` operator works in `bash`. Moreover, I asked myself, "Could I get that operator or others like it to work in Python?". So I set out to see if I could accomplish just that.

Let's really drill down on what I am hope to accomplish. Consider the following Python code:

```python

def square(x):
    return x * x

def inc(x):
    return x + 1

result = 2 | square | inc | square

print(f"Result: {result}")

```

Now the above Python code will not execute. (Be care as f-strings are a new feature so you will need a newer version of Python to hope to accomplish anything.) Go ahead, try it... I'll wait...

![Mr. Bean Waiting](/blog/images/waiting.jpg)

Didn't work did it? However, can we leverage Python and some of its magic methods to make it work? I am envisioning the following:

```python
from pipeliner import pipeliner

@pipeliner("|")
def square(x):
    return x * x

@pipeliner("|")
def inc(x):
    return x + 1

result = 2 | square | inc | square

print(f"Result: {result}")

```

The above is the fancy syntax for decorators in Python. However, think of those as functions that produce new functions with modified behavior. You know what... maybe we should take a quick side quest!

### Side Quest: Decorators
The purpose of decorators in Python is to add new behavior to an original function. (**Careful**: I realize that decorators are a little more complicated than that, but for our purposes we will stay with that idea).

One of the first times that I wrote a decorator of my own was for function timing. Let's see my __naive__ implementation.

```python
from time import perf_counter, sleep
from random import randint
from functools import wraps

# That is some good decorator right there... :P
def timeit(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
	result = func(*args, **kwargs)
	end = time.perf_counter()
	print(f"{func.__name__} took {end - start} seconds.")
	return result
    return wrapper


@timeit
def square(x):
    sleep(randint(0, 5))
    return x + 1

```

The above will time the function `square` and print how long it took to stdout. I added the `sleep` in order to see a meaningful measurement. (That highlights a shortcoming of this decorator, I only measure in whole seconds. We will have to fix that another time.)

Now we could go deep into decorators, but for now I think that is a good enough for our purposes. We don't want to pick up too much loot from our side quest that would prevent us from continuing the main quest. Which reminds me...

#### Back to the Main Quest!

Wooo, that was a nice little detour. However, let's dig in and find out how we are going to compose those pipelines of functions. Well, we need to talk about some of the interfaces that Python conforms to or the **Python Data Model**. In Python, there are magic methods that dictate how a Python object behaves when it is part of an expression that includes operators. Let's look at some examples:


```python

class Sample:
    def __init__(self, val):
        self.val = val
    def __add__(self, other):
        return Sample(self.val + other.val)
    

x = Sample(10)
y = Sample(3)
z = x + y

print(f"z: {z}") # Sample(13)

```

In the above sample code, I have overloaded the `+` operator (in one particular manner as magic methods require a little more to make work well). Now many of the operators in Python have similar methods such as `__sub__` overloads the `-`, `__mul__` overloads `*`, etc. Moreover, there are usually symmetric magic methods such as `radd`, `rmul`, `rsub`, etc. that reverse the argument order (think `2 + 3` vs `3 + 2`). Additionally, there are some operators such as `|` and `>>` that have magic methods associated with them as well. Ah ha! We can use those to achieve our goal. So we could do the following:

```python

class Pipe:
    def __init__(self, func):
        self.func = func
    def __rrshift__(self, parameter): # overload of '>>' on the right side
        return self.func(parameter)

def _square(x):
    return x * x

def _inc(x):
    return x + 1


square = Pipe(_square)
inc = Pipe(_inc)

result = 2 >> square >> inc

print(f"Result: {result}")


```

Run it! Go ahead... I'll wait again. 

.
.
.
.
It ran didn't it!

![Squirrel Victor](/blog/images/victory.jpg)

Now at this point, we can write a decorator that will allow us to choose an operator and then return the object (such as Pipe) that will allow our "wonderful" syntax.

Let's take a crack at it.
```python

def call(obj, parameter):
    return obj.func(parameter)

def pipeliner(operator):
    def inner(func):
	if operator == ">>":
            setattr(Pipe, "__rrshift__", call)
	if operator == "|":
	    setattr(Pipe, "__ror__", call)
        return Pipe(func)
    return inner

# so now we can do the following

@pipeliner("|")
def square(x):
    return x * x

@pipeliner("|")
def inc(x):
    return x + 1

result = 2 | square | inc | square

print(f"Result: {result}")

```

We are victorious! If you want to play around and see the repo that helped me make this blog post, then check out https://github.com/justin1dennison/pipeliner. I will warn you, there are a few things different, but everything works the same. Let me know what you think! See you around!


