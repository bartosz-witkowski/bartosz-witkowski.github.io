---
layout: wyas
title: Typeclasses and Show
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

Typeclasses and `show` 
======================

In this part we'll start implementing our evaluator. As a first step
we're going to just print out what was parsed.

New concepts 
============

Typeclass
---------

I've
[written](http://like-a-boss.net/2013/03/29/polymorphism-and-typeclasses-in-scala.html)
about typeclasses in scala before so if you *do* know scala but don't
know typeclasses you can take a peek.

We won't cover every nook and cranny of typeclasses but I'll try to
present the general gist of them. Typeclasses are a mechanism for
compile time predicate dispatch. That sounds loaded but is actually
quite simple - typeclasses allow you to define a set of predicates and
functions that can be reused with multiple types - example application:
operator overloading (though operator overloading is not implemented
that way in mercury).

Concretely, let's consider this simple typeclass:

{% highlight prolog %}
:- typeclass show(T) where [
  pred show(T, string),
  mode show(in, out) is det
].
{% endhighlight %}

The syntax for typeclasses is the `:-` token which start almost all
"type" related declarations the `typeclass` keyword followed by a comma
separated list of parameters, followed by the `where` keyword.
Surrounded by the `[` and `]` tokens is a comma departed list of
predicates or functions.

A typeclass is a promise that some types may wish to hold. In our
example you may want enforce the `show` promise. The `show` promise is
simple - for some type `T` we want its `string` representation.

Let's implement such a promise for the simples case of all - `string`.
`show`ing a string will unify the string with that string quoted:

{% highlight prolog %}
:- instance show(string) where [
  (show(String, "\"" +++ String ++ "\""))
]
{% endhighlight %}

Here you can see the **instance declaration** - sometimes called *the
evidence that `string` has `show`* (or is/supports `show`). The other
possibility is to create the implementation in a separate predicate e.g
`show_string` and use:

{% highlight prolog %}
:- instance show(string) where [
  pred(show/2) is show_string.
].
{% endhighlight %}

The `pred(show/2)` is mercury's way to disambiguate between predicates,
and functions and also between functions and predicates of different
arity. We will use this notation `foo/2` to always mean a `foo`
predicate with 2 arguments - unless explicitly stated it is a function.

We can use the `show` typeclass by implementing a simple `show` function
(typeclasses, functions, and predicates have a separate namespace so
there's no ambiguity):

{% highlight prolog %}
:- func show(T) = string <= show(T).
show(T) = String :- show(T, String).
{% endhighlight %}

The function declaration can be read as *forall T such that T is `show`
define a function from T to string*. At this stage it is important to
discern between the usage and the declaration - the *forall* word is key
here. This predicate "works" for all types that support show - even
those that are not written yet!

If you're familiar with java or c++ or language that support
`interfaces` the above concept is one of the key to understanding the
differences between typeclasses and interfaces. The other big difference
is that the there is no ["dynamic
dispatch](http://en.wikipedia.org/wiki/Dynamic_dispatch) if the compiler
can't find the evidence that some type supports a given typeclass it
won't compile. Example:

{% highlight prolog %}
main(IO_1, IO_Last) :-
  io.write_string(show("1"), IO_1, IO_2),
  io.write_string(show(1), IO_2, IO_Last).
{% endhighlight %}

Because there's no *evidence* that int supports the `show` typeclass
mercury will fail to compile this class.

Typeclasses and Modules 
-----------------------

Public typeclasses (i.e those that should be visible outside of the
module that they are declared) should be declared in the interface
section.

If you want to include a typeclass instance only a "stub" should be
exported i.e:

{% highlight prolog %}
:- instance show(string).
{% endhighlight %}

will export the `show` instance for string. The actual instance should
be implemented in the `implementation` section.

Matching and Multiple Predicate Rules 
-------------------------------------

Previously we discussed matching in this form:

{% highlight prolog %}
 ( X = option_1
   ...
 ; X = option_2
   ...
 ; X = option_N
   ...
 )
{% endhighlight %}

But there's also an alternative that is sometimes useful. For example
consider this adt:

{% highlight prolog %}
:- type foo ---> bar ; baz.
{% endhighlight %}

And the predicate

{% highlight prolog %}
:- pred show_foo(foo::in, string::out) is det.
show_foo(Foo, String) :-
  ( Foo = bar
    String = "bar"
  ; Foo = baz
    String = "baz"
  ).
{% endhighlight %}

We can also write it using multiple rules this way:

{% highlight prolog %}
:- pred show_foo(foo::in, string::out) is det.
show_foo(bar, "bar").
show_foo(baz, "baz").
{% endhighlight %}

Which at least for me is somewhat nicer. This will only work if the
match is complete (you cover all cases) - otherwise the predicate will
not be infered as `det`.

Goal
====

1) Create the `show` module:
  * Add the `show` typeclass discussed above - the typeclass declaration should be in the
    `interface` section.
  *  Export the `show` function implemented above and add it's implementation.
  *  Add instances of `show(string)` and `show(integer)` use `integer.to_string(integer) = string`.
  *  Add an `unwords` predicate and function. `unwords` should unify a list of string to a single string
concatenating them with spaces (`" "`).

{% highlight prolog %}
:- func unwords(list(string)) = string.
:- pred unwords(list(string)::in, string::out) is det. 
{% endhighlight %}

2) In the `adts` module add an instance of `show(lisp_val)` - to make it easier create a predicate:

{% highlight prolog %}
:- pred show_lisp_val(lisp_val::in, string::out) is det. 
{% endhighlight %}

On:
* `lisp_string(String)` you should make use of `show(string)`
* `lisp_number(Number)` use `show(integer)`
* `lisp_atom(Atom_Name)` unify the output string with the `Atom_Name`
* `lisp_boolean(yes)` should unify the output string with `"#t"`
* `lisp_boolean(no)` should unify the output string with `"#f"`
* `lisp_list(List)` should unify to a string that was constructed by wrapping parens around an `unwords`-ed list of strings, in which each element was evaluated recurisively by `show_lisp_val` i.e with an input argument `lisp_list([lisp_atom("a"), lisp_number(integer(5))])` the output string should be `(a 5)`
* `lisp_dotted_list(List, Last)` should unify with a string in which the first part should be shown the same way as `lisp_list(List)` was followed by a `" .  "` followed by the result of show\_lisp\_val(Last) i.e for `lisp_dotted_list([lisp_atom("a"), lisp_number(integer(5))], lisp_boolean(yes))` the output should be `"(a 5 . #t)"`

3)  In the `main` module:
  * if the parser doesn't match or matches partially "Cannot parse expression" should be printed. 
  * If it can then the output should be `show`ing that `lisp_val`.

Testing
=======

Check out if this works:

	$ ./main a
	a
	$ ./main '$'
	$
	$ ./main '(a$ b c . d)'
	(a$ b c . d)
	$ ./main '"foo"'
	"foo"

Implementation
==============

Full code for this part can be found
[here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/08).

Let's start from the changes in `main` module - first we have to add two
imports to `adts` and `show`, then in the `main` predicate modify what
happens after `string.to_char_list(First, CharList)`:

{% highlight prolog %}
( parser.top_level_expression(Exp, CharList, Rest) ->
  ( Rest = [] -> 
    io.write_string(show(Exp) ++ "\n", IO_2, IO_Last)
  ; io.write_string("Cannot parse expression!\n", IO_2, IO_Last))
; io.write_string("Cannot parse expression!!\n", IO_2, IO_Last))
{% endhighlight %}

Just a couple of changes and nothing fancy going on here - let's tackle
the `show` module starting with the `interface` section.

{% highlight prolog %}
:- module show.

:- interface.

:- import_module string, integer, list.

:- typeclass show(T) where [
  pred show(T, string),
  mode show(in, out) is det
].

:- func unwords(list(string)) = string.
:- pred unwords(list(string)::in, string::out) is det.

:- func show(T) = string <= (show(T)).

:- instance show(integer).
:- instance show(string).
{% endhighlight %}

After importing `string`, `integer` and `list` modules we export the
`show` typeclasss declaration, the `unwords` function and predicate
along with instances of `show` for `integer` and `string` types.

In the implementation section we can add instances of `show(integer)`
and `show(string)` and add the implementation of the `show(T)` function:

{% highlight prolog %}
:- implementation.

:- instance show(integer) where [
  (show(Int, String) :- String = integer.to_string(Int))
].

:- instance show(string) where [
  (show(String, "\"" ++ String ++ "\""))
].

show(T) = String :- show(T, String).
{% endhighlight %}

The only kind of complicated thing is the `unwords` predicate let's
break it down:

{% highlight prolog %}
unwords([], "").
{% endhighlight %}

Ok, so this is trivial - a empty `list` produces an empty `string` now
for an not empty list:

{% highlight prolog %}
unwords([Head|Tail], String) :-
  unwords_aux(Tail, Head, String).

:- pred unwords_aux(list(string)::in, string::in, string::out) is det.
unwords_aux([], Res, Res).
unwords_aux([Head|Tail], Acc, Res) :-
  unwords_aux(Tail, Acc ++ " " ++ Head, Res).
{% endhighlight %}

We use an auxiliary predicate with a accumulator passing style recursive
rule where in the iterative step we append the current element to the
accumulator.

Note that this may not be terribly efficient - it depends how `string`
is implemented in the worst case (where for example string is backed by
a string) this can be **very** inefficient.

Having done the `show` module, let's finish up the changes to the `adts`
module. In the interface section we only add one declaration:

{% highlight prolog %}
:- instance show(lisp_val).
{% endhighlight %}

And in the implementation section we add the instance:

{% highlight prolog %}
:- instance show(lisp_val) where [
  pred(show/2) is show_lisp_val
].
{% endhighlight %}

Let's go over `show_lisp_val` part by part:

{% highlight prolog %}
show_lisp_val(lisp_string(String), show(String)).
show_lisp_val(lisp_number(Integer), show(Integer)).
{% endhighlight %}

For `lisp_string` and `lisp_number` we just delegate to `show(string)`
and `show(integer)` respectively, going forwards:

{% highlight prolog %}
show_lisp_val(lisp_atom(Name), Name).
{% endhighlight %}

For `lisp_atom` we print out the atom name. Now `lisp_bool`:

{% highlight prolog %}
show_lisp_val(lisp_bool(yes), "#t").
show_lisp_val(lisp_bool(no),  "#f").
{% endhighlight %}

Note how we match both on `lisp_val` and `bool` here - we could rewrite
it so that matching on the boolean value be an inner match but this is
also possible.

Now for the trickier part:

{% highlight prolog %}
show_lisp_val(lisp_list(List), String) :-
  lisp_val_list_to_string_list(List, String_List),
  unwords(String_List, Single_String),
  String = "(" ++ Single_String ++ ")".

:- pred lisp_val_list_to_string_list(list(lisp_val)::in, list(string)::out) is det.
lisp_val_list_to_string_list(Val_List, String_List) :- 
  lisp_val_list_to_string_list_aux(Val_List, [], String_List).

:- pred lisp_val_list_to_string_list_aux(list(lisp_val)::in, list(string)::in, list(string)::out) is det.
lisp_val_list_to_string_list_aux([], Acc, list.reverse(Acc)).
lisp_val_list_to_string_list_aux([H | T], Acc, String_List) :- 
  show_lisp_val(H, String),
  lisp_val_list_to_string_list_aux(T, [String | Acc], String_List).
{% endhighlight %}

Let's go over `lisp_val_list_to_string_list` first - again we
recursively match on the input list this time a list of `lisp_val`s - in
each iteration step we convert the `Head` to a `string` using
`show_lisp_val` and we append the `String` to the accumulator. In the
base case we **reverse** the list so that it comes "the right way".

Now `show_lisp_val(lisp_list(...))` `unwords` the list of strings into a
single list and adds parentheses to both sides.

`show_lisp_val` for `dotted_list` is not much different:

{% highlight prolog %}
show_lisp_val(lisp_dotted_list(List, Last), String) :-
  lisp_val_list_to_string_list(List, String_List),
  unwords(String_List, List_String),
  show_lisp_val(Last, Last_String),
  String = "(" ++ List_String ++ " . " ++ Last_String ++ ")".
{% endhighlight %}

The only thing that differs is that we add the dot `.` and the last
element (also converted to string using `show_lisp_val`.

In the next part we will learn about higher order predicates and
anonymous predicates and functions, so that we can start building our
evaluator!

[Next](09-beginnings-of-an-evaluator.html)
