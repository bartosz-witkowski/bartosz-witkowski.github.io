---
layout: wyas
title: A Simple Parser
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

A simple parser 
===============

Believe it or not with what we know we're almost ready to start writing
a simple parser

The goal 
--------

1)  We want to define a predicate that recognizes the following chars:
    "!\$%&\|\*+-/:\<=\>?@\^\_\~\#"

2)  the predicate should be named `symbol` and its type and mode
    signatures should look like:

{% highlight prolog %}
:- pred symbol(char, list(char), list(char)).
:- mode symbol(out, in, list(char)::out) is semidet. 
{% endhighlight %}

 -   the first output value should unify to the matched symbol
 -   the second output value should unify to the "rest of" the
     characters (what remains to be parsed after the `symbols` parser
     unified).

3)  in `main` take the first command line argument and checks if the
    `symbol` parser matches it \* if the first command line parameter
    doesn't exist print out an error and exit (do nothing).

4)  if the parser (`symbol` predicate) matches (unifies) the input
    string we should print out "Found value: " *the value we found*, if
    not it should print out "No match."

**Note**: while it may seem arbitrary structuring the arguments this way
has a purpose - this will be explored further in the series.

New concepts 
============

`string`s and `char`s {#s_and_s}
---------------------

Previously we've discussed `int`s and `list`s - now to actually
implement parts of the scheme interpreter we need to introduce two new
data types `char` and `string`.

While both `char` and `string` specifics are implementation dependent
(they're backed by `char` and a `char*` in the C grade and `Char` and
`String` in java) we can think of `char` as a UTF [code
point](http://en.wikipedia.org/wiki/Code_point) and a `string` as a
container of chars (whether this is a list, an array or something else
entirely is implementation specific, and uninteresting).

To use `chars` or `strings` in your code (in type declarations, or using
predicates from them you must first import the module).

{% highlight prolog %}
:- import_module char, string.
{% endhighlight %}

If you don't export any predicate (and you shouldn't!) that has char or
string you should put the above imports into the **implementation**
section (and not to the **interface** section)

More On Lists 
-------------

What I've been glossing over so far is that `list` in mercury is
actually `list(T)` - lists in mercury are defined for any (the
parametric type `T` is meant to be similar to a variable name) type i.e
we can have an `list(int)`, `list(char)`, `list(string)` etc.

Lists in mercury are exactly like the list you may know from other
languages - each cell or element of the list points to the rest of the
list (or is an empty list).

An empty list in mercury is just `[]` and we can deconstruct a list to
its head and tail like so:

{% highlight prolog %}
[Head | Tail] = List.
{% endhighlight %}

This will unify `Head` with the first element of the list **or** fail if
the list is empty. If `Head` will be unified then `Tail` will always
unify to the rest of the list (the whole list sans the first element).

If we're not interested in some variable (anywhere in our predicate not
only in lists) we can name it either `_` or any name that starts with an
underscore. That variable is considered "anonymous". The mercury
compiler will warn us if we have unused variables - so in the case we
really don't care about them it's good form to make them anonymous.

We'll need the above concepts to get the first argument given to the
program (and don't forget to `:- import_module list`!)

Getting command line arguments {#getting_command_line_arguments}
------------------------------

The `io` module provides an useful predicate for that:

{% highlight prolog %}
:- pred io.command_line_arguments(list(string)::out, io::di, io::uo) is det.
{% endhighlight %}

Mercury has some syntax sugar for predicates with only one mode - the
variable modes and determinism can be added directly to the type
declaration as long as every variable has a mode (separated by `::`).

The above predicate will (always - hence `det`... just a reminder) unify
to the argument list "consuming" the `di` `io` instance and providing a
new one.

Useful methods on strings {#useful_methods_on_strings}
-------------------------

To get a list of `char`s from a string use:

{% highlight prolog %}
:- pred string.to_char_list(string::in, list(char)::out) is det.
{% endhighlight %}

There's also one predicate that may come in in handy - checking if a
`string` contains a `char`:

{% highlight prolog %}
:- pred string.contains_char(string::in, char::in) is semidet.
{% endhighlight %}

This predicate will unify if the input 'string' contain the input
'char'.

If statements {#if_statements}
-------------

Mercury like most languages has a if statement built-in, the syntax is:

{% highlight prolog %}
( condition ->
  consequent 
; alternative)
{% endhighlight %}

Or using a more "literal" style:

{% highlight prolog %}
if condition then
   consequent else 
   alternative
{% endhighlight %}

While I prefer the second style I'll use the first throughout they
tutorial because: 

1. It's non-ambiguous and doesn't have to be
additionally parenthesized like the second construct. 
2. It seems more ubiquitous both in prolog and in mercury code.

The if then else expression will try to unify `consequent` if
`condition` unifies or `alternative` if it does not.

If you're coming out of a functional programming background you may
think that **if**s in mercury are expressions returning a value - but
that isn't the case - the whole model of logic programming doesn't
understand what "returning a value" is - there's only unification.

One caveat of using if expressions is that you can't instantiate
something in the condition for example introduce a variable - it has to
be done in the consequent and alternative instead (and this is how we
can "conditionally" pass some values). Consider:

{% highlight prolog %}
( 1 = 1 ->
  X = "Hello world",
; X = "Goodbye world"),
io.write_string(X, IO_In, IO_out).
{% endhighlight %}

This small snippet would print out "Hello world" or, if we change the
condition from `1 = 1` to `1 = 2` it would print "Goodbye world".

This on the other hand would not compile:

{% highlight prolog %}
( 1 = 1, X = "Hello world" ->
  io.write_string("1 = 1", IO_1, IO_2)
; io.write_string("1 != 1", IO_1, IO_2),
  io.write_string(X, IO_2, IO_3)).
{% endhighlight %}

Though referencing X is completely valid **inside** the if statement.

Testing
=======

This is how our program should behave:

{% highlight shell %}
$ ./main 
No arguments given!
$ ./main a
No match!
$ ./main '>'
Found match: >
$ ./main '$'
Found match: $
{% endhighlight %}

Tips
====

If you're trying to implement this by yourself the things that tricky
might be tricky are:

-   module imports - if you use `char`s, `list`s, or `string`s (and you
    should!) - import them in the implementation section.
-   compiler errors - try to read what the compiler is telling you and
    not glaze over its output - while sometimes the error messages may
    seem nonsensical (like forgetting a `.` after a predicate
    declaration) most of them will pinpoint your error.
-   `io` - remember the `main` predicate should unify some "clean" input
    `io` instance and the `io` instance that has gathered all effects,
    this may mean that you should construct your `main` predicate in the
    following way:

{% highlight prolog %}
main(IO_1, IO_Last) :-
   io.write_string("Hello", IO_1, IO_2),
   Some_Variable = "world",
   io.write_string(Some_Variable, IO_2, IO_3),
   % we can print chars too!
   io.write_char('C', IO_3, IO_Last).
{% endhighlight %}

Note the new `io.write_char` predicate.

Implementation
==============

Code for this part is located
[here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/04)

Let's get the obvious imports end exports out of the way:

{% highlight prolog %}
:- module main.

:- interface.

:- import_module io.

:- pred main(io, io).
:- mode main(di, uo) is det.

:- implementation.

:- import_module char, string, list.
{% endhighlight %}

Now for the `symbol` predicate:

{% highlight prolog %}
:- pred symbol(char::out, list(char)::in, list(char)::out) is semidet.
symbol(C, ListIn, ListOut) :-
    ListIn = [C | ListOut],
    string.contains_char("!$%&|*+-/:<=>?@^_~#", C).
{% endhighlight %}

There's really nothing tricky behind it. You might wonder about
`ListIn = [C | ListOut].` but that's equivalent to
`ListIn = [C | X], X = ListOut.` We consume one character (the first
one), and unify `ListOut` with what's unconsumed.

The predicate may obviously fail on trying to unify `C` with the head
(or the fist element) of `ListIn` - but that's ok - well handle that in
main. Next we just check if `C` is any of the chars that we wanted to
match by using `contains_char` (and that may also fail to unify).

Lastly let's look at the `main` predicate:

{% highlight prolog %}
main(IO_1, IO_Last) :-
    io.command_line_arguments(Arguments, IO_1, IO_2),
    ( Arguments = [First | _Rest] ->
      string.to_char_list(First, CharList),
      ( symbol(C, CharList, _Other_Chars) ->
        io.write_string("Found match: ", IO_2, IO_3),
        io.write_char(C, IO_3, IO_4),
        io.write_string("\n", IO_4, IO_Last)
      ; io.write_string("No match!\n", IO_2, IO_Last))
    ; io.write_string("No arguments given!\n", IO_2, IO_Last)).
{% endhighlight %}

Again nothing tricky here - first we use `command_line_arguments` from
the `io` module then we check using the *if-then-else* statement if the
`Arguments` have a first element - if not we print "No arguments
given!\\n")

If so we convert the `First` element of the list into `CharList` a list
of characters and then again check if
`symbol(C, CharList, _Other_Chars)` succeeded - if so we print out what
we've matched if not we print out "No match!\\n".

You might think that programming in mercury is a little cumbersome with
all the `io` variables being passed around - don't fret mercury has
special mechanisms that alleviate this - we will introduce them in due
time.

End
===

That's all for this part - we'll introduce recursion and extend our
simple parser to handle whitespace. See you!

[Next](05-recursion-and-parsing-whitespace.html)
