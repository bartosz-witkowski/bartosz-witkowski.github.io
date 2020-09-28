---
layout: wyas
title: Algebraic Data Types-and the Base of our Interpreter
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

Algebraic Data Types and the Base of Our Interpreter 
====================================================

In this part we will define the core data types used in the interpreter
such as the strings, numbers, atoms etc.

Algebraic Data Type 
===================

Sum types, disjunct types, tagged unions or variant types... are
actually a pretty simple concept - a value that is a sum type may be
"matched" (deconstructed) against it's possible constructors.

Consider a simple boolean sum type:

{% highlight prolog %}
:- type boolean ---> yes ; no.
{% endhighlight %}

The syntax is pretty easy `boolean` is the name of the sum type and on
the right from the arrow (note the number of `-` in it) there's a
`;`-delimited list of constructors.

Now a predicate that operates on booleans may use this the following
way:

{% highlight prolog %}
:- pred say(boolean::in, string::out) is det.
say(Boolean, String) :-
  ( Boolean = yes,
    String = "YES :)!"
  ; Boolean = no,
    String = "NO! ;("
  ).
{% endhighlight %}

The `( X = constructor1 ; X = constructor2 ; ... ; X = constructorN )`
construct is how mercury **matches** on values. Matches are guaranteed
to be exhaustive (check every possible constructor) - not matching on
some value is a compile time error.

As a sidenote - `;` in mercury or prolog is the
[disjunction](http://en.wikipedia.org/wiki/Logical_disjunction) - you've
seen it used in if-then-else and now in sum types and matches. It may be
easier to understand if you know to read the types as "yes OR no",
"Boolean unifies with yes, String unifies with "YES :)!", OR ..." or in
if-then-else "Some condition THEN something OR something else".

Constructors in sum types may be parametrized by values e.g:

{% highlight prolog %}
:- type shape ---> circle(float) 
                 ; square(float) 
                 ; triangle(float, float, float)
                 .
{% endhighlight %}

Now consider this `area` predicate, for example on how to match
constructors with parameters:

{% highlight prolog %}
:- pred area(shape::in, int::out) is det.
area(Shape, Area) :-
  ( Shape = circle(Radius)
    Area = math.pi * Radius * Radius
  ; Shape = square(Side)
    Area = Side * Side
  ; Shape = triangle(A, B, C)
    P = (A + B + C) / 2
    Area = math.sqrt(P * (P - A) * (P - B) * (P - C))
  ).
{% endhighlight %}

Like everywhere if we don't care about a variable we can name it either
`_` or give it a `_` prefix.

Now algebraic data type are sum types that can also be recursive - the
standard example is the list:

{% highlight prolog %}
:- type list(T) ---> nil ; cons(T, list(T)).
{% endhighlight %}

Note that this list definition is generic - types, like predicates and
functions, can also have generic parameters. So the above definition
states "A list `T` is either `nil` OR is `cons` containing a value and
another list `T`"

Multi module compilation 
========================

Before we only wrote in one (`main`) module from now on we'll divide the
project into multiple modules - for now it may not be so useful - but
organizing it this way will be less of a hassle in the long run.

Mercury comes out of the box with a nice build system in the form of
mmc. To compile a library use:

    $ mmc --make lib[LIBRARY_MODULE]

and the main module is just:

    $ mmc --make [MAIN_MODULE]

If they're in one directory mercury compiler will work out the
dependencies by itself.

`integer`s and `bool`s 
======================

`integer`s in mercury are like `int`s but with arbitrary precision -
we'll use them to represent lisp numbers.

`bool` is a type that can store boolean values - its constructors are
exactly like the ones we've seen in the adt examples:`yes` and `no`.

Goal
====

Create an `adts` module with the following declarations:

{% highlight prolog %}
:- type lisp_val ---> lisp_atom(string)
                    ; lisp_list(list(lisp_val))
                    ; lisp_dotted_list(list(lisp_val), lisp_val)
                    ; lisp_number(integer)
                    ; lisp_string(string)
                    ; lisp_bool(bool.bool)
                    .
{% endhighlight %}

For some motivation:

1.  In scheme atoms are one of the two fundamental data type (the other
    being the list) - you can think of them as a string with an unique
    identity (two atoms with the same name are always equal).
2.  Lists are *THE* data structure that comes to my mind when I think
    about Lisp (it's LISt Processing for a reason) - we'll back one up
    will a mercury list.
3.  A dotted list represents the scheme form `(a b . c)` - we store the
    all elements "untill the dot" in a list and after the dot in another
    field.
4.  A number backed by the `integer` type.
5.  A string backed by the `string` type.
6.  A boolean backed by the `bool` type.

Implementation
==============

The implementation for this part is available here:
`https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/06`

It's not that exciting honestly - the adt is spelled out, and everything
was explained above. So let's see the two changed files.

adts.m:

{% highlight prolog %}
:- module adts.

:- interface.

:- import_module string, list, integer, bool.

:- type lisp_val ---> lisp_atom(string)
                    ; lisp_list(list(lisp_val))
                    ; lisp_dotted_list(list(lisp_val), lisp_val)
                    ; lisp_number(integer)
                    ; lisp_string(string)
                    ; lisp_bool(bool.bool)
                    .
{% endhighlight %}

And the Makefile:

{% highlight make %}
all:
	mmc --make libadts
	mmc --make main
{% endhighlight %}

Outro
=====

In the next part we'll add parsers returning the values above and
introduce a way to make writing/reading parsers a little easier.

[Next](07-dcgs-and-parsing-continued.html)
