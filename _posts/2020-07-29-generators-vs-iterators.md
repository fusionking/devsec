---
title:  "Python Generators vs Iterators"
date:   2020-07-20 12:34:48 +0300
categories: python
author: "Can"
excerpt: "Learn the hidden gems of python generators and lists"
header:
    overlay_image: /assets/images/programming.jpg
    overlay_filter: linear-gradient(rgba(2, 0, 36, 0.5), rgba(0, 146, 202, 0.5))
slug: generators-vs-iterators
sidebar:
    nav: docs
layout: custom
---

One of the most commonly asked questions when it comes to python is the difference
between `generators` and `iterators`.

First of all, both `generators` and `iterators` allow you to iterate over a collection of elements one by one.

Any object which has a `__next__` method implemented on its class, is an iterable object.
So, an `iterator` is a producer which yields successive values from its iterable object.
The `__next__` method has to return an element based on the managed state attributes of the class.
It should know which values have been obtained already, so when next() is called on the iterable, it knows what value to return next.

Lets clarify the concept with an example. Suppose you have a EvenRange class, which holds a minimum and maximum integer, which specify the range.
This class must yield the integers within the range by increments of 2. You define the class, and try to iterate over it:

{% highlight python %}
class EvenRange:
    def __init__(self, _min, _max):
        self.min = _min
        self.max = _max
evenrange = EvenRange(40, 60)
for number in evenrange:
    print(number)
{% endhighlight %}

However, this will yield to an error like this:

```TypeError: 'EvenRange' object is not iterable```

Because we haven't defined this object to be an iterable, you encounter this error.
So we define an iterator class, which is returned from EvenRange's `__iter__` method:

{% highlight python %}
...
def __iter__(self):
    return EvenRangeIterator(self)
{% endhighlight %}

{% highlight python %}
class EvenRangeIterator:
    def __init__(self, obj):
        self.obj = obj
        self.pos = self.obj.min
    def __next__(self):
        if self.pos < self.obj.max:
            result = self.pos
            self.pos += 2
            return result
        else:
            raise StopIteration
{% endhighlight %}

The `__next__` method is called whenever the code encounters an iterator expression such as a for loop. The for loop calls the `__next__` method in every iteration,
and updates the `pos` instance variable until we reach the maximum range. The loop is terminated if a StopIteration is raised.

<hr>

`Generators` are special kind of functions which return a lazy iterator. Lazy iterators are iterators, which yield the requested element when it is actually needed. Therefore, lazy iterators do not store the element's content in memory.

You have to use a `yield` statement to return the generator object.
`Yield` statements tell the caller where a value is sent back and remembers the **state** of the function, so successive calls of the same generator function will remember the previous yielded value.

A famous example of a generator function could be the case when you want to generate an infinite sequence of numbers:

{% highlight python %}
def infinite_sequence():
    num = 0
    while True:
        yield num
        num += 1
{% endhighlight %}

If you call this within a for loop, the program will run infinitely until you stop it via an interruption.
However, if you call `next` on this method, you will see that at each call, it will yield a number and successive calls remember the yielded previous numbers.
So, with this method, you do not actually use every number in the infinite sequence: you use a **selected set of consecutive numbers** to process. Simply put, you 
use the element when it is **actually needed**.

So this was all for today! Enjoy the rest of your day!