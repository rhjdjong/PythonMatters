---
title: "Everything Is An Object, part 2"
date: 2019-11-20
excerpt: "Exploring the Python object model: Object type"
---

[PLR]: https://docs.python.org/3/reference/index.html
[Part1]: {% post_url 2019-11-18-Everything-is-an-object-part-1-identity %}
[Part3]: {% post_url 2019-11-20-Everything-is-an-object-part-3-value %}

## Introduction

This is the second blog about the Python object model.
In the [previous blog][Part1] I noted
that the [Python Language Reference][PLR]
states that

> Every object has an identity, a type and a value.

I also noted that this is an ambiguous statement,
because the verb "to have" can mean so many different
things.

In this blog I investigate the meaning of the
phrase "Every object has a type".
The verb "has" in this phrase
indicates a relationship.

The Language Reference states:

> An object’s type determines the operations that the object supports
(e.g., “does it have a length?”)
and also defines the possible values for objects of that type.

So the object’s type determines the behavior of the object.
That means that the object’s *methods* are defined
in the object’s type.
The phrase *and also defines the possible values
for objects of that type*
in the language reference is also ambiguous.
It refers to the rather nebulous concept of *value*,
which is the subject of the next section.
As I wrote in the [previous post][Part1]
this series of blogs started when I was
thinking about what happens when I create
a subtype of `int` that only accepts
values from 1 through 6.
The `__new__`  method of this subclass
was responsible for ensuring that only these
values were accepted.
In this case, the fact that the type defines
the possible values is straight-forward.
But in general the concept of value
is not so easy to grasp. The [next blog
post][Part3] in this series deals exclusively
with the values of objects.

## Types

The `type(x)` built-in function returns
the type of object `x`.
Nothing magical happens here,
the `type()` function simply
returns whatever is stored in the `__class__` attribute
of object `x`.

By the way,
this highlights an important difference between Python on the one hand
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

Since an object's type is available as an attribute
of that object,
we can &mdash; in principle &mdash;
manipulate the object's type
by modifying the object's  `__class__` attribute.
The [Python Language Reference][PLR] however states that an
object's type is unchangeable, with a footnote that
states that in some cases it is possible to change an object's
type, but that it can easily lead to strange behaviour.

And since everything in Python is an object,
an object’s type is itself an object.
This means that the object's type also has a type,
and the type of the object's type is also an object and has a type,
and so on and so forth.

Coming back to the phrase I quoted in the [previous post][Part1],
there is another way to read it: "In Python, everything is an `object`"
(note the font change).

In Python, `object` is the root object of the entire object hierarchy.
Every class in Python has an attribute `__bases__` that
lists the superclasses of the class, i.e.
the classes from which the class is derived.
For `object`, and only for `object`, the `__bases__` attribute
is an empty tuple:

    >>> object.__bases__
    ()
    >>>

Every other class is derived, directly or indirectly
from `object`.
And since an instance of a class is also an instance
of the superclasses of that class,
this means that every object in Python is, directly or indirectly,
an instance of the class `object`.

Python has two built-in functions that deal with
object instances and subclasses:

* `isinstance(x, C)` is true if 
  `x` is an instance of the class `C`, i.e.
  if the type of object `x` is (a subclass of) the class `C`.
* `issubclass(y, C)` is true if
  `y` is a class that is (a subclass of) the class `C`.

This means that for every object `x` in Python,
`isinstance(x, object)` is true.
Earlier we saw that `type(x)` gives us the type of the object `x`.
So another statement that is always true is `issubclass(type(x), object)`.
But since the type of an object is also an object, `isinstance(type(x), object)`
must also be always true.
So for every object, the type of that object is both an instance and a subclass of `object`.

To clarify this, suppose we have a class `A`.
All classes derive, directly or indirectly,
from `object`, so `A` is a subclass of `object`.
So we have:

![A is a subclass of object](/assets/Everything-is-an-object/object-class-inheritance.svg){: .align-center}

An instance `a`  of that class is, by definition, an instance of `A`.

![a is an instance of A](/assets/Everything-is-an-object/instance-class.svg){: .align-center}

And class `A` is also an object, so it is an instance of yet another class,
and that class must again be a subclass of `object`:

    >>> type(A)
    <class 'type'>
    >>> issubclass(type(A), object)
    True
    >>> type(A).__bases__
    (<class 'object'>,)
    >>> 

![class A is an instance of type](/assets/Everything-is-an-object/class-metaclass.svg){: .align-center}

Combining all the above, we get the following diagram:

![instance a, class A, object, and type](/assets/Everything-is-an-object/instance-class-metaclass.svg){: .align-center}

Now the class `object` of course also has a type:

    >>> type(object)
    <class 'type'>
    >>> 

![object is an instance of type](/assets/Everything-is-an-object/object-metatype.svg){: .align-center}

So, `type` is the class of both `A` and `object`.
In fact, in Python the class of a class is `type` 
(or, in some cases, a subclass of `type`).
Such a class of a class is called a _metaclass_.
You may wonder where this ends.
Do we also have classes of classes of classes?
Well, it ends right here, because
type is a subclass of `object`, so it also
has `type` as its class:

    >>> type(type)
    <class 'type'>
    >>> type is type(type)
    True
    >>> 

If we now look again at the diagram with `a`, `A`, `object`, and `type`,
we can replace `A` with `type` (because `type` is a subclass
of `object`), and we can replace `a` with `object`
(because `object` is an instance of `type`).

![object is an instance of type large picture](/assets/Everything-is-an-object/object-type-metatype.svg){: .align-center}

Now the two occurrences of `object` in reality depict
the same object but in different roles.
Similarly the two occurrences of `type` also refer
to one and the same `type` object.
Re-arranging this so that both `object` and `type` are
shown only once,
we get the following diagram for the relationships
between `object` and `type`:

![object is an instance of type condensed](/assets/Everything-is-an-object/object-type.svg){: .align-center}

We see that the "is instance of" relationship is circular:
`object` is an instance of `type`
(because `object.__class__` is `type`),
and `type` is an instance of `object`
(because `type.__class__` is `type`,
and `type` is a subclass of `object`).

The "is subclass of" relationship goes only one way though:
`type` is a subclass of `object`, but not the other way around.

Confusing, to say the least.
 
Note that this self-referential relationship
between `object` and `type` is built-in in the language.
It is not possible to create such loops with user-defined classes.

### Classes and instances

There are of course many ways in which you can
partition the different kinds of objects
that exist in Python.
The above shows one such partition:
an object is either a class (meaning it can be instantiated)
or a non-class.

Examples of non-class objects are e.g. integer objects (`5`),
strings (`"hello"`), lists (`[1, 2, 3]`), mappings (`{'a': 1, 'b': 2}`)
but also functions, modules and various other object instances.

Class objects act as factories for other objects. `int` produces
integer objects, `str` produces string objects, `list` produces
list objects, etc.
The class `type` is also a factory, but now for class objects.
In fact, without this class there could not be any
user-defined classes.

And similar to the way in which a class controls the
behaviour of the objects that it generates, do
metaclasses (`type` and its subclasses) control
the behaviour of a class, in particular its creation.

The division between classes and non-classes is
quite noticeable.
If you try to ask if the instance `a` (of class `A`)
is a _subclass_ of object, you get:

    >>> a = A()
    >>> issubclass(a, object)
    Traceback (most recent call last):
      File "<pyshell#3>", line 1, in <module>
        issubclass(a, object)
    TypeError: issubclass() arg 1 must be a class
    >>> 

So the Python runtime can determine whether an object
is a class or a non-class.
The [language reference][PLR] is silent on this
subject, which means that this is really an
implementation detail.
But at least in CPython 3.8 it is the
presence of the `__bases__` attribute
that decides if an object is considered a class &mdash;
at least for the `issubclass()` function.

    >>> a.__bases__ = (object,)
    >>> issubclass(a, object)
    True
    >>>

It is fun to play around with this, and try to fool
the Python runtime.
You can even pretend that you can create a class
that is not a subclass of `object`:

    >>> a.__bases__ = ()
    >>> issubclass(a, object)
    False
    >>> 

You could try to continue this and see
if you can really create a class
that is not derived from `object`.
But at some point,
probably sooner than later,
you will run into
various safeguards built in the Python runtime
that prevent this.
But hey, it's fun to try and see how far you can get.

### Summary

To summarize:

* The type of an object is stored in its `__class__` attribute;
* Every class is (directly or indirectly) a subclass of `object`;
* Every class is (directly or indirectly) an instance of `type`;
* At the top of the class hierarchy, `object` and `type` have a self-referential relationship
that is built-in in the language.
