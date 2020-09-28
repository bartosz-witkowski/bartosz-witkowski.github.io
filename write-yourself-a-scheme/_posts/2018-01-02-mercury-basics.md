---
layout: wyas
title: Mercury Basics
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

Mercury basics
==============

Execution model 
===============

To have an understanding what mercury and logic programming is about
I'll try to introduce a mental model of execution of mercury programs.
If you're familiar with how logic programming works (e.g. by knowing
prolog) you can safely skim this section.

Because functional programming and logic programming are so closely
related I'll try to explain logic programming using functional
programming. If you don't know functional programming this may be little
off-putting but I hope that the descriptions are good enough that you
will be able to get a feeling for logic programming.

So, if you *are* familiar with functional programming you probably know
that the main execution model is lambda calculus. I won't try to
introduce such a formal model for logic programming but one 
does exists - instead I'll talk about something called **equational reasoning**.

In a functional programming language we can **always** think of the
program as a step of substitutions. Using Lisp as an example the output
of the following program:

{% highlight scheme %}
(append (list 1 2 3) (append (list 4 5) (list 6 7)))
{% endhighlight %}

is equivalent to this program:

{% highlight scheme %}
(append (list 1 2 3) (list 4 5 6 7))
{% endhighlight %}

and to this program:

{% highlight scheme %}
(list 1 2 3 4 5 6 7)
{% endhighlight %}

I will talk more about Lisp in the series but for now let's stop here.

Hopefully you see how each of the steps relate to each other. This
process is not necessarily how the computer will interpret this program
but the results will **always** be equivalent.

In logic programming we also use equational reasoning to think about the
program execution. But logic programming adds another dimension to this
process.

Logic programs are based on the concept of **unification** (trying to
make "two sides equal"), a logic program consists of two things:

1.  Facts
2.  Query

One can view execution of a **functional** program as a list of
expressions undergoing simplification. A **logic** program is a list of
facts and a query.

The query and facts can be thought as the "opposite sides of an
equation". The act of trying to check if both sides are equal **is** the
program.

This seems to imply two things:

1.  By only changing the query (and not the facts) you can get a
    completely different program.
2.  Word like "trying" and "querying" imply that a program can fail.

Indeed both above statements are true.

The first point is seen as a strength of logical programming where
programs can be used in multiple ways - we will see what this means
shortly.

The second point is also important - some completely normal logic
programs will fail to unify. For a brain dead example:

{% highlight prolog %}
X = 1, X = 2, X.
{% endhighlight %}

Will fail to unify. ',' in both mercury and prolog is just ∧
(conjunction) from mathematics. Logical variables always start with an
Upper case letter or an underscore `_`.

The facts for the above program are:

1.  X = 1
2.  X = 2

At a glance we see that the facts given are nonsensical. Querying for X
must fail to unify.

Here we see the additional dimension I mentioned earlier - apart from
thinking about simplifying terms we also must think about if they fail
or not.

Variables
=========

For simplification I'll say that a logical variable can be in two states

1.  **ground** - having a set value and immutable
2.  **free** - yet unset and "waiting" for a value

More about unification
======================

Unification works by trying to make "two sides" equal. For now I'll list
a couple of rules of unification (there are more but they will be
introduced as we go):

1.  two ints unify only if they are equal
2.  lists will unify with each other if all of the elements of the list
    unify (position-wise)
3.  two ground variables will unify only if their values will unify
4.  a free variable will always unify with a ground variable, grounding
    it with that value.
5.  two free variables technically **cannot** unify - this differs from
    prolog where two free variables will become aliased. Aliasing
    "entwines" two variables so that unifying one unifies with the other
    aliased one - i.e `X = Y, Y = 2` will unify both `X` and `Y` to `2`.
    As long as all the variables are ground in the scope of one
    predicate (see below) we can think of them as aliased. What will
    happen behind the scenes is that mercury will reorder the facts so
    that the "groundings" happen first.

Quiz
----

Which of the examples will unify:

1.  `1 = 1.`
2.  `A = B, A = 1.`
3.  `A = B, A = 1, B = 2.`
4.  `A = B, A = 1, B = C, C = 2.`
5.  `[1, 2] = [1, 2].`
6.  `[2, 1] = [1, 2].`
7.  `X = 1, X = 2.`
8.  `X = 1, 2 = Y.`
9.  `X = Y, Y = X, X = [1, 2].`

Answers:

1.  Y
2.  Y
3.  N
4.  N
5.  Y
6.  N
7.  N
8.  Y
9.  Y

Predicates
----------

We can decompose functional programs into smaller units called...
functions.

A function of arity $$n$$ is a relation between $$n$$ 
input arguments and the output. The output should be the
same for the same inputs. Or more formally: 

$$
  x_1 = y_1, x_2 = y_2, \ldots x_n = y_n \Rightarrow f(x_1, x_2, \ldots, x_n) =
f(y_1, y_2, \ldots,
y_n)
$$

A **predicate** is the smallest unit of decomposition available to the
logical programmer. 

A predicate of arity $$n$$ - is a relationship between between $$n$$ variables
and *true*, *false* - the result of unification.

Predicates can have **input** and **output** variables - this differs
from functions that only have input variables and one (implicit) output.

An output variable will be **free** before unifying with the predicate
and **ground** after.

An input variable will be **ground** (because we must "know" it at
unification time) and **ground** after (in other word it won't change
it's ground state).

While mercury can sometimes infer if the variable is an input variable
or an output one we will be explicit about it during this tutorial. To
mark some variables as input we use a mode declaration in the form of:

{% highlight mercury %}
:- mode predicate_name(in, out). 
{% endhighlight %}

Where `:-` is a token that starts the mode declaration, `mode` is the
keyword, predicate\_name is the programmer-readable name of the
predicate followed by a comma separated list of modes. There are more
modes then just `in` and `out` and they will be introduced further along
in the series.

Knowing all this we can rewrite the above scheme program to mercury:

{% highlight prolog %}
% mode declaration
:- mode not_hello_world(out).
% facts
not_hello_world(X) :- 
  list.append([4, 5], [6, 7], A),
  list.append([1, 2, 3], A, X).
% query
not_hello_world(Z), Z.
{% endhighlight %}

First of all a `%` starts a comment in mercury - everything until the
end of the line will be ignored.

Predicates consist of two parts - the head:

{% highlight prolog %}
not_hello_world(X)
{% endhighlight %}

which consists of the predicate name (the programmer readable name of
the predicate) and a comma separated list of variables - in this case we
only have one variable called X. The names of the variables don't matter
outside of the scope of the predicate.

The second part of the predicate is the predicate body, in this case:

{% highlight prolog %}
  list.append([4, 5], [6, 7], A),
  list.append([1, 2, 3], A, X)
{% endhighlight %}

Predicates are separated from the body by the `:-` token (sometimes
called the neck) and terminated by the `.` token.

The predicate body is is composed (by conjunction or disjunction) of
smaller facts or other predicates.

To interpret the above program we need to know that:

1\. `list.append(A, B, AB)` - unifies `AB` to with concatenation of lists
`A` and `B` - when used in the `(in, in, out)` mode. 1. `[1, 2, 3]` is a
[list comprehension](http://en.wikipedia.org/wiki/List_comprehension)

We can model the computation in our head by a series of steps

{% highlight prolog %}
not_hello_world(Z), Z. 
% let's expand what not_hello_world stands for
not_hello_world(Z) ---> 
list.append([4, 5], [6, 7], A), list.append([1, 2, 3], A, X) -->
% We can potentially simplify either
%  (1) list.append([4, 5], [6, 7], A) or 
%  (2) list.append([1, 2, 3], A, X) 
% for (2) we don't know how list.append(in, out, out) works - that means we
% can only simplify it after A is ground we can however simplify (1)
A = [4, 5, 6, 7], list.append([1, 2, 3], A, X) --->
list.append([1, 2, 3], [4, 5, 6, 7], X),
X = [1, 2, 3, 4, 5, 6, 7]
% comming back to our query we have:
X = [1, 2, 3, 4, 5, 6, 7], X.
% which is true
{% endhighlight %}

The predicate above will unify `Z` with a list - again while mercury can
infer the type of the variable it is good form to write it out
explicitly:

{% highlight prolog %}
:- pred not_hello_world(list(int)).
{% endhighlight %}

This declares the `not_hello_world` predicate to unify with a list of
ints. We will learn more about types further along in the series.

Now for an example of a query which will fail:

{% highlight prolog %}
not_hello_world(Z), Z = [1, 2, 3].
{% endhighlight %}

We would expand `not_hello_world(Z)` as before and "ground" it to the
value `[1, 2, 3, 4, 5, 6, 7]` - our query would fail however because we
cannot unify ` [1, 2, 3, 4, 5, 6, 7]` with `[1, 2, 3]`.

This query on the other hand is not possible at all:

{% highlight prolog %}
Z = [1, 2, 3], not_hello_world(Z).
{% endhighlight %}

Why? Terms in mercury are by default unified top to bottom (from `main`), and from left to right (
[see](https://www.mercurylang.org/information/doc-latest/mercury_ref/Semantics.html#Semantics)
not\_hello\_world has only the `not_hello_world(out)` mode
declared so we cannot use a ground variable. We need to declare another
mode for `not_hello_world(Z)`:

{% highlight prolog %}
:- mode not_hello_world(in).
{% endhighlight %}

That will actually fail to compile - because not\_hello\_world(in) can
fail the possibility of failing must be explicitly marked. The correct
mode declarations are actually:

{% highlight prolog %}
:- mode not_hello_world(out) is det.
:- mode not_hello_world(in) is semidet.
{% endhighlight %}

**det** and **semited** are two determinism modes that we will use for
the time being. If a predicate is declared as `det` it means that it
**cannot** fail. If there's a possibility that the predicate *could*
fail with this determinism declaration the compiler will signal an error
(at compile time).

`semidet` predicates can either fail (to unify) or will succeed (unify)
- just like `not_hello_world(in)` which will fail to unify for
infinitely many int lists but will unify with the list
`[1, 2, 3, 4, 5, 6, 7]`

Quizz
-----

Try to infer all posible mode and type declarations for the following
predicates:

1.
{% highlight prolog %}
q1(A, B, C) :-
  A = 5,
  B = A,
  C = [B].
{% endhighlight %}

2.
{% highlight prolog %}
q2(A, B, C) =
  A = [B, C],
  B = 1,
  C = 1.
{% endhighlight %}

3.
{% highlight prolog %}
q3(A, B) =
  A = 5,
  B = list(list(A)).
{% endhighlight %}

Answers:

1.

{% highlight prolog %}
:- pred q1(int, int, list(int)).
:- mode q1(out, out, out) is det.
:- mode q1(in, out, out) is semidet.
:- mode q1(out, in, out) is semidet.
:- mode q1(out, out, in) is semidet.
:- mode q1(in, in, out) is semidet.
:- mode q1(in, out, in) is semidet.
:- mode q1(out, in, in) is semidet.
:- mode q1(in, in, in) is semidet.
{% endhighlight %}

2.

{% highlight prolog %}
:- pred q2(list(int), int, int).
:- mode q2(out, out, out) is det.
% all modes with at least one **in** argument will be semidet; same as in q1
{% endhighlight %}

3.

{% highlight prolog %}
:- pred q3(int, list(list(int))).
:- mode q3(out, out) is det.
% all modes with at least one **in** argument will be semidet; same as in q1 and
% q2 but only for two variables
{% endhighlight %}

Closing words
=============

In practice you cannot write programs like `X = 1, X = 2, X.` - while
this is style of querying is possible in the prolog console mercury
lacks that mechanism.

Nevertheless this method of analyzing mercury programs is applicable
everywhere. Check out the next part for our very first mercury program.

[Next](03-hello-world.html)
