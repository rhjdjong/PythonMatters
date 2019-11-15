---
title: "Everything Is An Object"
date: 2019-03-01
---

## Introduction

The other day I needed to write
a Python integer subclass that only allowed
a certain range of values, say 1 through 6. Such a class is easy to implement:
create a subclass of `int`, and override the `__new__()` method to check that the
value of the instance is in the allowed range.
But then I thought about the internals of what happens when you
create e.g. an instance with the value 5.
And I realised that the fact that value of the object (5 in this case)
can only be determined by comparing it for equality with a Python integer with the value 5.
The actual "fiveness" of the object
(the bit pattern that defines the value)
is hidden by the language.

Another way to phrase this is
that Python does not allow direct access
to the underlying hardware.
Whenever you try to do so, Python wraps the result
in a Python object.
And there is no way from within the Python language
to pierce through this wrapper,
and access the underlying data structures in
the computer's memory.
Of course, there are modules that allow you to
manipulate the underlying hardware. The `ctypes`
module provides just such an interface.
But notice that `ctypes` and similar modules are
extension modules, not part of the language itself.
And even with these modules, you are still presented
with Python objects that represent the
underlying structure.

This is just one example where I realized that the commonly used phrase
"In Python, everything is an object" has more implications than just the fact
that you can pass around functions and types at will.
In this post I look a closer look at the meaning
and implications of this phrase,
and find it is [objects all the way](https://en.wikipedia.org/wiki/Turtles_all_the_way_down),
both up and down.

[PLR]: https://docs.python.org/3/reference/index.html
The main source for this article is the [Python Language Reference for Python 3][PLR].
That means that other versions of Python (in particular the Python 2.x versions)
may have slightly different behaviors.

## Objects

The language reference contains a chapter on the [data model](https://docs.python.org/3/reference/datamodel.html)
that describes objects:

> Objects are Python’s abstraction for data.
All data in a Python program is represented by objects or by relations between objects.
(In a sense, and in conformance to Von Neumann’s model of a “stored program computer,”
code is also represented by objects.)

That an object can represent data is fairly obvious:
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

### Identity

The identity of an object is like a social security number:
it is a bookkeeping device used by the Python runtime
to uniquely identify each object that exists.

The identity of an object is time-wise unique.
It is set when the object is created, and it never changes during the object’s lifetime.
That means that at any given moment in time,
there can never be two different Python objects with the same identity.
But the object’s identity can be re-used after the object is garbage-collected,
hence time-wise unique.
Note that the object’s identity is *not* the object’s name.
There can be multiple names that refer to the same object – or no name at all. 
And at some later time, each and any of those names may refer to any other object.
The identity of an object remains the same though,
and is unique for that object during the object’s lifetime.

It is important to realize that the object’s identity
cannot be used or manipulated from within a Python program.
In other words, the identity of an object is itself *not* a Python object –
there is simply no way in Python-the-language to get hold of the identity of an object.
The closest you can get is an integer object that represents the identity,
as returned by the built-in function `id()`.
But note that this integer object is not the identity itself, only a representation of it.
In other words, you cannot take this integer value and use it to reach the object.

Most built-in functions work by calling a special method on the object the operate upon.
E.g. `len(x)` is equivalent to `x.__len__()`.
But not so with the built-in function `id()`.
When you call `id(x)`, you actually ask the Python runtime to return
the integer representation of `x`'s identity -- the object `x` itself has
no influence over the result of this function call.
In fact, an object does not even know its identity, only the Python runtime knows it.

The usefulness of the built-in function `id()` is therefore limited.
As stated earlier, the identity of an object can be reused after the object has been destroyed.
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
<code>a&nbsp;is&nbsp;b</code> gives `True` if `a` and `b` refer to the same object, and `False` otherwise.
Again, this is handled completely by the Python runtime.
The objects themselves have no way to influence the outcome of the `is` operator.

So, to summarize:
* The identity of an object is a bookkeeping device for the Python runtime.
* The identity is not an object, and hence cannot be manipulated from within a Python program.
* The built-in function `id()` and the operator `is`
are handled completely by the Python runtime, without any possibility for the objects
concerned to influence the result.

### Type

The Language Reference states:

> An object’s type determines the operations that the object supports
(e.g., “does it have a length?”)
and also defines the possible values for objects of that type.

So the object’s type determines the behavior of the object.
That means that the object’s *methods* are defined in the object’s type.
The `type(x)` function returns the type of object `x`.
Nothing magical happens here, the `type()` function simply
returns whatever is stored in the `__class__` attribute of object `x`.
And since everything in Python is an object,
an object’s type is itself an object.
The phrase *and also defines the possible values for objects of that type*
in the language reference is also ambiguous.
It refers to the rather nebulous concept of *value*,
which is the subject of the next section.

Note that "an object has a type" has a different meaning for "has" than
"an object has an identity". As I discussed above, the identity is not intrinsic
to the object, but a bookkeeping device used by the Python runtimee.
But for the type, "has" implies a relationship with another object.
That means that an object can - in principle - manipulate its type,
by modifying its `__class__` attribute.
This highlights an important difference between Python on the one hand
and e.g. C++ and Java on the other hand:
the type of an object is associated with the object itself,
not with the name (variable) you use to refer to that object.
Any name can reference an object of any type,
and at a later time that same name may reference another object with a completely different type.
In other words, like C++ and Java, Python is a strongly typed language,
often times even more so than C++ and Java.
But unlike C++ and Java, which are statically typed languages,
Python is dynamically typed, which means that a name (variable)
can reference differently typed objects over time.

Coming back to the phrase I quoted in the beginning,
there is another way to read it: "In Python, everything is an `object`"
(note the font change).
In Python, `object` is the root object of the entire object hierarchy.
That means that every object in Python is, directly or indirectly,
an (instance of the class) `object`.

Python has two built-in functions that deal with object instances and subclasses:
* `isinstance(x, C)` is true if the type of object `x` is (a subclass of) the class `C`.
* `issubclass(y, C)` is true if `y` is a class that is (a subclass of) the class `C`.

This means that for every object `x` in Python, `isinstance(x, object)` is true.
Earlier we saw that `type(x)` gives us the type of the object `x`.
So another statement that is always true is `issubclass(type(x), object)`.
But since the type of an object is also an object, `isinstance(type(x), object)`
must also be always true.
So for every object, the type of that object is both an instance and a subclass of `object`.

When we apply this to `object` itself, we see something interesting:

    >>> type(object)
    <class 'type'>
    >>> type(type)
    <class 'type'>
    >>>

The type of `object` is the class `type`. And the type of class `type` is again class `type`.
So `type` is an instance of itself:

    >>> isinstance(type, type)
    True
    >>> 
    
But every object is an instance of `object`, so class `type` must also be an instance of `object`:

    >>> isinstance(type, object)
    True
    >>> 

But an object is also an instance of its type, so:

    >>> isinstance(object, type)
    True
    >>> 

And because `type` is a subclass of `object`, `object` is also an instance of itself:

    >>> isinstance(object, object)
    True
    >>> 

Similar equalities do not hold for `issubclass`:

    >>> issubclass(type, object)
    True
    >>> issubclass(object, type)
    False
    >>> 
    
So subclassing is strictly top to bottom; you cannot have loops in subclass relations.
But this strict hierarchy does not hold for instances, at least not for `object` and `type`.
The following class diagram illustrates this.

![Object-type relationships](/assets/Everything-is-an-object/object-type.svg)

Note that this self-referential relationship is built-in in the language.
It is not possible to create such loops with user-defined classes.

To summarize:

* The type of an object is stored in its `__class__` attribute,
* Every class is a subclass of `object`
* At the top of the class hierarchy, `object` and `type` have a self-referential relationship
that is built-in into the language.

### Value

Of the three concepts *identity*, *type*, and *value*, the value is in a way the most basic,
and at the same time the most difficult concept to comprehend.
The identity of an object can be understood as a bookkeeping device.
The type of an object `x` can be retrieved by `type()`, and is stored 
in an attribute `__class__` of the object.
But how do you get at the *value* of `x`? Where is it stored? What is it?

The Python reference manual has some handwaving about values:
sometimes it is considered to be the result of evaluating an expression,
which of course it quite literally is:
"to evaluate" means "to determine the value"
For structured objects, like lists and class instances,
the value of the object can be understood
to be the value of the objects that it contains.
The value of a list instance is the sequence of
values of the objects contained in the list.
The value of a class instance is the value of its attributes.

But what about elementary type instances without internal structure?
Consider an int object `x` with the value 5:

	>>> x = 5
	>>> id(x)
	1720829744
	>>> type(x)
	<class 'int'>
	>>> x
	5
	>>>

At first sight, it might seem that the value of the object `x` is the object `x` itself.
That would make sense, because the value of an object is data
that can be used and manipulated in a Python program.
And since all data in Python is an object,
the value of x must also be an object.
But even for elementary types like `int`s, this is
not true.
Consider:

    >>> a = 2000
    >>> b = 1500 + 500
    >>> 

Now, it is obvious that `a` and `b` have the same value.
    
    >>> a == b
    True
    >>> 

But they are _not_ the same object:

    >>> a is b
    False
    >>> 

As I mentioned in the introduction,
the only way to learn something about the
value of an `int` object is to compare it
with the value of another object.
And equality comparisons use the object
type's `__eq__()` and `__ne__()` special methods.
So, in a very real sense, the value of an object
is determined by the class definition.
This is essentially the meaning of the phrase we
encountered earlier:

> An object’s type ... defines the possible values for objects of that type.

There is something unsatisfying about this, though.
Objects can have attributes that influence
their behavior, but that do not play a role
in determining whether two objects are equal:

    >>> class A:
        def __init__(self, x, y):
            self.x = x
            self.y = y
        def __eq__(self, other):
            return self.x == other.x
        def hi(self):
            if self.y < 4:
                print('hi')
            else:
                print('howdy')
    
                
    >>> a = A(1, 3)
    >>> b = A(1, 5)
    >>> a == b
    True
    >>> a.hi()
    hi
    >>> b.hi()
    howdy
    >>> 
    
This is of course a contrived example.
But if the different behaviors
are relevant for your use
of this class, then you would not consider
`a` and `b` to be equal.
It seems fair to say that the
value of an object is actually dependent on the
circumstances in which that value is used.
For two objects `x` and `y` to be equal,
<code>x&nbsp;==&nbsp;y</code> is a necessary condition,
but it may not be sufficient.
Or in other words: it depends...

Looking back, I think that the conclusion must
be that an object's value is not an object
in Python.
Rather it is an intrinsic _quality_ of an object.
When we say that an object _has_ a value,
the meaning of _has_ is similar to saying
that a knife _has_ a sharp edge.
Having a sharp edge is what gives a knife its value,
but you cannot separate that sharp edge from the
knife and treat it as a separate entity.
Similarly, you cannot take an object's value out of
the object, and treat it as a stand-alone object.
The only way to use an object's value
is to use the object itself.

In the beginning of this section I said that
the [Python Lnaguage Reference][PLR] has only
some broad statements and hand-waving when it
comes to defining an object's value.
With this investigation I hope that I have
added some insights as to why that is,
in addition to the unavoidable additional
hand-waving that I have added myself.

