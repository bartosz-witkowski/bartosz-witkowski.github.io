---
layout: wyas
title: Beginnings of an Evaluator
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

Beginnings of an Evaluator 
==========================

In this part we will take care of the necessities needed to actually
start writing the first parts of an evaluator. Nothing complicated
happens in this part but this will setup the groundwork for the later
parts. Let's get started then!

New concepts 
============

Aliasing/unification expressions 
--------------------------------

An expression of the form `X @ Y` is equivalent to the expression
`Z = X, Z = Y`.

Why is this valuable? We can use it in matches to make the code more
readabe:

{% highlight prolog %}
:- type foo ---> bar(int, int) ; baz(int, int, int) ; qux(string, string).

:- pred match_foo(foo::in, string::out) is det.
match_foo(Bar @ bar(_, _), String) :- do_something_with_bar(Bar, String).
match_foo(Baz @ baz(_, _, _), String) :- do_something_with_baz(Bar, String).
match_foo(Qux @ foo(_, _), String) :- do_something_with_qux(Qux, String).
{% endhighlight %}

Here the `Bar`/`Baz`/`Qux` are *aliases* to what's on the other side of
the `@` operator.

Goal
====

The goal of this part is to implement these two predicates:

1.  `eval(lisp_val::in, lisp_val::out) is semidet` such that the output
    value is

    -   for `lisp_string` the input value
    -   for `lisp_number` the input value
    -   for `lisp_boolean` the input value
    -   for `lisp_list([atom("quote"), X])` is X.
    -   and the predicate should fail to unify for everything else.

2.  `eval` should be in a module named `eval`.

3.  `read_expr(list(char)::in, lisp_val::out) is semidet` which for now
    should just use with `parser.top_level_expression` where the `Rest`
    is an empty list

The `main` predicate should use `read_expr` and either output "Cannot
read expression.", "Cannot eval expression." or `show` the `eval`-ed
output.

Testing
=======

{% highlight prolog %}
$ ./main "'atom" 
atom
$ ./main 2
2
$ ./main "\"a string\""
"a string"
$ ./main "(- -)"
Cannot eval expression!
$ ./main "5 (1)"
Cannot parse expression!
{% endhighlight %}

Implementation
==============

The full source code for this part can be found
[here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/09).

The implementation for this part of the tutorial is nothing to write
home about - very simple and easy to understand.

Let's start with the `eval` module:

{% highlight prolog %}
:- module eval.

:- interface.

:- import_module adts.

:- pred eval(lisp_val::in, lisp_val::out) is semidet.

:- implementation.

:- import_module list.

eval(X @ lisp_number(_), X).
eval(X @ lisp_string(_),  X).
eval(X @ lisp_bool(_), X).
eval(lisp_list(List), Out) :-
  List = [lisp_atom("quote") | Tail],
  Tail = [Out].
{% endhighlight %}

Like I said - nothing complicated - just remember how the `@` operator
works. Now for the `main` module:

{% highlight prolog %}
    ( read_expr(CharList, Exp) ->
      ( eval(Exp, Evaled) ->
        io.write_string(show(Evaled) ++ "\n", IO_2, IO_Last)
      ; io.write_string("Cannot eval expression!\n", IO_2, IO_Last))
    ; io.write_string("Cannot parse expression!\n", IO_2, IO_Last))
{% endhighlight %}

Again, nothing tricky. This was way to easy - so in the next installment
we'll introduce a couple of new concepts, see you soon!

[Next](10-higher-order-predicates.html)
