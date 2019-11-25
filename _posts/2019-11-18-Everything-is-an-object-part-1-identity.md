---
title: "Everything Is An Object, part 1"
date: 2019-11-20
excerpt: "Exploring the Python object model: Object identity"
---

[PLR]: https://docs.python.org/3/reference/index.html
[Part2]: {% post_url 2019-11-20-Everything-is-an-object-part-2-types %}
[Part3]: {% post_url 2019-11-20-Everything-is-an-object-part-3-value %}

## Introduction

The other day I needed to write
a Python integer subclass that only allowed
a certain range of values, say 1 through 6.
Such a class is easy to implement:
create a subclass of `int`,
and override the `__new__()` method to check that the
value of the instance is in the allowed range.
But then I thought about the internals
of what happens when you
create e.g. an integer instance with the value 5.
And I realised that value of the object (5 in this case)
can only be determined by comparing
it for equality with another Python integer
that has the value 5.
The actual "fiveness" of the object
(the bit pattern that defines the value)
is hidden by the language.

Another way to phrase this is
that Python does not allow direct access
to the underlying hardware.
You can try to do so, but the Python runtime
always wraps the result
in a Python object.
There is just no way from within the Python language
to pierce through this wrapper,
and access the underlying data structures in
the computer's memory.
There are, of course, modules that allow you to
manipulate the underlying hardware. The `ctypes`
module provides just such an interface.
But notice that `ctypes` and similar modules are
extension modules, not part of the Python language itself.
And even with these modules, you are still presented
with Python objects that represent the
underlying structure.

This is just one example where I realized
that the commonly used phrase
"In Python, everything is an object"
has more implications than just the fact
that you can pass around functions and types at will.
In this series of posts I take a closer look at the meaning
and implications of this phrase,
and find it is
[objects all the way](https://en.wikipedia.org/wiki/Turtles_all_the_way_down),
both up and down.

The main source for this article is
the [Python Language Reference for Python 3][PLR].
That means that other versions of Python
(in particular the Python 2.x versions)
may have slightly different behaviors.

The language reference contains a chapter
on the [data model](https://docs.python.org/3/reference/datamodel.html)
that describes objects:

> Objects are Python’s abstraction for data.
All data in a Python program is represented by objects or by relations between objects.
(In a sense, and in conformance to Von Neumann’s model of a “stored program computer,”
code is also represented by objects.)

That an object can represent data is obvious:
an integer objects represents an integer number,
a string object represents a Unicode string,
and a code object represents executable code.
Data can also be represented by relations between objects.
The meaning of that phrase is perhaps not immediately clear.
But think of a list.
A `list` object has links (relations) to the objects it contains.
Those links, i.e. the fact that the objects are present in the list,
and their order in the list, represent data that is not present in
the individual objects themselves.

The language reference continues with:

> Every object has an identity, a type and a value.

A simple, obvious, and very ambiguous statement.
Ambiguous, because the 
English verb "to have" is ambiguous.
It can refer to ownership (I have a car),
it can refer to a relationship (I have a mother),
it can refer to labels that other people or institutions
assign to us (I have a name, I have a social security number),
to name just a few of the possibilities.

While working on this post, I constantly encountered things
that I wanted to include.
As a consequence, the original blog post was becoming
way too big, at least for my taste.
I therefore decided to split this in three separate
blog posts, to keep the size of the individual
articles manageable.
This post deals mainly with an object's identity.
Other posts will deal with an object's [type][Part2],
and an object's [value][Part3].

## Object Identity

The identity of an object is like a social security number:
it is assigned and managed by the Python runtime,
and it is essentially a bookkeeping device
used by the runtime
to identify and keep track of each object that exists.

The identity of an object is time-wise unique.
It is set when the object is created, and it never changes during the object’s lifetime.
That means that at any specific moment in time,
there can never be two different Python objects with the same identity.
But the object’s identity can be re-used after the object is garbage-collected,
hence time-wise unique.
Note that the object’s identity is _not_ the object’s name.
There can be multiple names that refer to the same object &mdash; or no name at all. 
And at some later time, each and any of those names may refer to any other object.
The identity of an object remains the same though,
and is unique for that object during the object’s lifetime.

Because the identity of an object is essentially
a bookkeeping device for the Python runtime,
the object’s identity simply
cannot be used or manipulated from within a Python program.
In other words, the identity of an object is itself *not* a Python object –
there is just no way in Python-the-language to get hold of the identity of an object.
The closest you can get is an integer object that represents the identity,
as returned by the built-in function `id()`.
But note that this integer object is not the identity itself, only a representation of it.
In other words, you cannot take this integer value
and use it to reach back to the object.

Most built-in functions work by calling a special method on the object the operate upon.
E.g. `len(x)` is equivalent to `x.__len__()`.
But not so with the built-in function `id()`.
When you call `id(x)`, you actually ask the Python runtime to return
the integer representation of `x`'s identity.
The object `x` itself has
no influence over the result of this function call.
In fact, an object does not even know its identity, only the Python runtime knows it.

The usefulness of the built-in function `id()`
is therefore limited.
Because the function `id()` gives a representation of the
identity of the object
*at the time that the function was called*,
you cannot reliably store this value and
use it later to compare it with the result
of another `id()` call.
This is because,
as I mentioned earlier,
the identity of an object can be reused after the object has been destroyed.
This can happen even within one Python statement:

	>>> id(object()), id(object())
	(1641220969264, 1641220969264)
	>>>

What happens here is that an instance of the class `object` is created,
and the representation of its identity is retrieved.
The first `object` instance is then garbage-collected
(because there are no further references to it),
and the second `object` instance is created.
The Python runtime in this case elects to re-use the same identity as for the previous instance.

To demonstrate that this is not an artifact caused by the use
of the same class for both objects, the following gives similar results:

	>>> class A: pass

	>>> class B: pass

	>>> id(A()), id(B())
	(1641262059472, 1641262059472)
	>>>

Note that this behavior is _not_ guaranteed by the language.
Other Python implementations may keep the individual object instances alive longer,
or elect to never re-use identities.
But the above shows that it is not safe to use the `id()` function to determine if objects are identical:
the integer value returned by `id()` can be obsolete by time you want to use it.
 
To compare objects for identity you must use the `is` operator:
<code>a&nbsp;is&nbsp;b</code> gives `True` if
and only if `a` and `b` refer to the same object, and `False` otherwise.
Again, this is handled completely by the Python runtime.
The objects themselves have no way to influence the outcome of the `is` operator.

So, to summarize:

* The identity of an object is a bookkeeping device for the Python runtime.
* The identity is not an object, and hence cannot be manipulated from within a Python program.
* The built-in function `id()` and the operator `is`
are handled completely by the Python runtime, without any possibility for the objects
concerned to influence the result.
