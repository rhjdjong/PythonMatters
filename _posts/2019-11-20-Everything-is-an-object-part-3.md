---
title: "Everything Is An Object, part 3"
date: 2019-11-20
excerpt: "Exploring the Python object model: Object value"
---

## Introduction

[PLR]: https://docs.python.org/3/reference/index.html
[Part1]:{% post_url 2019-11-18-Everything-is-an-object-part-1-identity %}
This is the second blog about the Python object model.
In the [previous blog][Part1] I noted
that the [Python Language Reference][PLR]
states that

> Every object has an identity, a type and a value.

I also noted that this is an ambiguous statement,
because the verb "to have" can mean so many different
things.

In the [previous post][Part1] I noticed
that the meaning of "has" in the phrase
"Every object has an identity"
indicates that an external
agency (the Python runtime) assigns an identity
to every object, in order to identify and keep track
of all objects.

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
is not so easy to grasp. The next blog
post in this series deals exclusively
with the values of objects.

The `type(x)` built-in function returns
the type of object `x`.
Nothing magical happens here,
the `type()` function simply
returns whatever is stored in the `__class__` attribute
of object `x`.
And since everything in Python is an object,
an object’s type is itself an object.

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

Since an object's type is available as an attributef that object, we can
That means that an object's type can &mdash; in principle &mdash;
be manipulated by modifying the object's  `__class__` attribute.

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

Confusing, to say the least. Let me try to make this somewhat clearer.
The `bases` class attribute
basically shows the direct superclass(es) of the class:
We see that `object` has no superclasses,
so object is not a subclass of any other class.

    >>> object.__bases__
    ()
    >>>
   
But `type` is really a direct subclass of `object`:

    >>> type.__bases__
    (<class 'object'>,)
    >>> 
    
So subclassing is strictly top to bottom; you cannot have loops in subclass relations.
But this strict hierarchy does not hold for instances, at least not for `object` and `type`.
If you have a class `A`,
then an instantiation of that class `a`  is (by definition)
an instance of `A`. 
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

