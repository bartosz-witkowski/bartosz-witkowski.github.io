---
layout: wyas
title: Recursion and Parsing Whitespace
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

Recursion and Parsing Whitespace
================================

Goal
----

In this part we'll extend our simple parser to match against symbols
preceded by arbitrary amounts of whitespace:

    $ ./main '      
    Found match: $

We'll use [BNF](http://en.wikipedia.org/wiki/Backus–Naur_Form) to
describe grammars from now on - if you don't know BNF check out the
above wikipedia entry and familiarize yourself with it.

The grammar we want to match is:

    rule ::= whitespace symbol
    whitespace ::= whitespace1 whitespace | ""
    whitespace1 ::= ' ' | '\t'

`whitespace1` is either a space or a tab. In general we won't worry
about new lines.

Functions
=========

Previously to print out a match we used this very convoluted
construction:

{% highlight prolog %}
io.write_string("Found match: ", IO_2, IO_3),
io.write_char(C, IO_3, IO_4),
io.write_string("\n", IO_4, IO_Last)
{% endhighlight %}

Using some `append` predicate for strings wouldn't be actually that much
better:

{% highlight prolog %}
% this is pseudocode mercury doesn't have a - `append(in, in, out) is det.` predicate
S_1 = "Found match: ",
append(S_1, C, S_2),
append(S2, "\n", S_3),
io.write_string(S3, IO_2, IO_Last)
{% endhighlight %}

For situations such as these you can use functions. Mercury **has** some
support for functional programming but it's embedded in the "logical
world" we discussed before.

By default functions in mercury behave just like `det` predicates but
have one special "output" argument. They aren't just syntax sugar -
functions occupy their own name space and you can have functions names
with the same arity (+1) as a predicate.

In general function declarations have the form:

{% highlight prolog %}
:- func function_name(t1, t2, t3, ..., tn) = to.
{% endhighlight %}

Where `t1`, .., `tn` - are input types (can be different, same as in a
predicate), and `to` is the output type. Functions of two input
arguments can also use an infix style - and that style is used in the
`++` function that can append two strings (from the `string` module):

{% highlight prolog %}
:- func string ++ string = string.
{% endhighlight %}

We can unify the output of a function with either on the left or the
right side:

{% highlight prolog %}
X = some_function(a, b, c),
some_function(a, b, c) = X.
{% endhighlight %}

So using `++` and another function from the `string` module
`char_to_string` we can rewrite the above to:

{% highlight prolog %}
  io.write_string("Found match: " ++ char_to_string(C) ++ "\n!", IO_2, IO_Last)
{% endhighlight %}

We will use functions to simplyfy the parser code.

Recursion
=========

To be able to parse one or more (or in this case 0 or more) characters
we need to introduce the concept of
[recursion](http://en.wikipedia.org/wiki/Recursion).

In logic programming as well as functional programming recursion
replaces the imperative notion of looping - it is important to remember
that recursion can represent everything that simple looping could.

For an example of recursion let's analyze a predicate calculating a
length of a list:

{% highlight prolog %}
:- pred list_length(list(A)::int, int::out) is det.
list_length(List, Size) :-
  ([X | Xs] = List ->
    list_length(Xs, Tail_Size),
    Size = Tail_Size + 1
  ; Size = 0).
{% endhighlight %}

`+` is a function defined on `int`s and it works exactly like you think
it should ;). The `A` in `list(A)` stands for a generic argument - the
list predicate should work for all types of lists.

Let's analyze how this works. First let's try unifying with an empty
list

{% highlight prolog %}
list_length([], Size) 
% -->
( [X | Xs] = [] ->
  list_length(Xs, Tail_Size),
  Size = Tail_Size + 1
; Size = 0)
% [X | Xs] = [] obviously fails so we unify:
Size = 0
{% endhighlight %}

Now for this list: `[a, b, c]`

{% highlight prolog %}
list_length([a, b, c], Size), Size
% -->
( [X | Xs] = [a, b, c] ->
  list_length(Xs, Tail_Size),
  Size = Tail_Size + 1
; Size = 0),
Size
% [X | Xs] = [a, b, c] unifies to X = a, Xs = [b, c] so:
list_length([b, c], Tail_Size),
Size = Tail_Size + 1,
Size.
% we must unify Tail_Size first:
( [X | Xs] = [b, c] ->
  list_length(Xs, Tail_Size_2), % name clash here - renamed the second argument
  Tail_Size = Tail_Size_2 + 1
; Tail_Size = 0),
Size = Tail_Size + 1,
Size.
% [X | Xs] = [b, c] unifies X with b and Xs with [c]
list_length([c], Tail_Size_2),
Tail_Size = Tail_Size_2 + 1
Size = Tail_Size + 1,
Size.
% We must expand list_length([c], Tail_Size_2)
([X | Xs] = [c] ->
  list_length([], Tail_Size_3),
  Tail_Size_2 = Tail_Size_3 + 1
; Tail_Size_2 = 0),
Tail_Size = Tail_Size_2 + 1
Size = Tail_Size + 1,
Size.
% [X | Xs] = [c] unifies X = c, Xs = []
list_length([], Tail_Size_3),
Tail_Size_2 = Tail_Size_3 + 1
Tail_Size = Tail_Size_2 + 1
Size = Tail_Size + 1,
Size.
% Again we expand list_length([], Tail_Size_3) - but we know that unifies
% Tail_Size_3 with 0 so:
Tail_Size_2 = 0 + 1
Tail_Size = Tail_Size_2 + 1
Size = Tail_Size + 1,
Size.
% 0 + 1 unifies to 1 -->
Tail_Size = 1 + 1
Size = Tail_Size + 1,
Size.
% 1 + 1 unifies to 2 ->
Size = 2 + 1,
Size.
% Unifies to
Size = 3.
{% endhighlight %}

Tail Recursion / Last Call Optimization 
---------------------------------------

You may have noticed that every expansion of the `list_length`
introduces a variable "lagging" behind waiting to be unified - this is
the unfortunate side-effect of structuring predicates this way. Those
variables will take up space on the stack and in extrema cases may blow
it.

There is a "trick" to make that work however, consider this two
predicate:

{% highlight prolog %}
:- pred list_length(list(A)::int, int::out) is det.
list_length(List, Size) :- 
  list_length_aux(List, 0, Size).

:- pred list_length_aux(list::in, int::in, int::out) is det.
list_length_aux(List, Acc, Size) :-
  ( List = [X | Xs] ->
    list_length_aux(Xs, Acc + 1, Size)
  ; Size = Acc).
{% endhighlight %}

The whole logic moved to the helper `list_length_aux` method - and there
the introduction of helper variables might seem weird but let's see this
at work:

{% highlight prolog %}
list_length([a, b, c], Size), Size.
% expanding list_length -->
list_length_aux([a, b, c], 0, Size), Size.
% --->
( [a, b, c] = [X | Xs] ->
  list_length_aux(Xs, 0 + 1, Size)
; Size = 0),
Size.
%  [a, b, c] = [X | Xs] unifies X with a, and Xs with [b, c] so
list_length_aux([b, c], 0 + 1, Size)
Size.
% -->
( [b, c] = [X | Xs] ->
  list_length_aux(Xs, 1, Size)
; Size = 1),
Size.
% [b, c] = [X | Xs] unifies X to b and Xs to [c]
list_length_aux([c], 1 + 1, Size),
Size.
% --->
( [c] = [X | Xs] ->
  list_length_aux(Xs, 2 + 1, Size)
; Size = 2),
Size.
% [c] = [X | Xs] unifies X with c and Xs with []
list_length_aux([], 2 + 1, Size)
Size.
% -->
( [] = [X | Xs] ->
  list_length_aux(Xs, 3, Size)
; Size = 3),
Size.
% [] = [X | Xs] will fail so:
Size = 3, Size.
{% endhighlight %}

As you see now we don't pollute the stack with unnecessary variables.
This technique is known in fp-circles as "tail call optimization" (TCO)
and in logic programming circles as "last call optimization (LCO).

If we can represent the recursion in a predicate in such a way that the
recursive unification is the last thing done (the predicate is not
unified) the mercury runtime will be able to "trim" away unneeded stack
so it will behave exactly like we have shown above.

This often needs the introduction of an auxiliary variable - sometimes
called the accumulator and (and this technique is sometimes called
accumulator passing style).

To ease the use of such predicates some "public" predicate most often
then not are used to (like `list_length`, and `list_length_aux` here).

You can view recursion like [mathematical
induction](http://en.wikipedia.org/wiki/Mathematical_induction) - to
help you get started think about "the base case" and the "next step".

Quiz
----

Implement all the following predicates using accumulator passing style,
if you think it is worth it implement using "simple recursion". It may
be worthwhile trying to analyze the predicates as I did above for
`list_length`.

### 1

{% highlight prolog %}
:- pred last(list(A)::in, A::out) is semidet.
{% endhighlight %}

Unifies with the last element of the list or fails on empty list.

### 2

{% highlight prolog %}
:- pred sum(list(int)::in, int::out) is det.
{% endhighlight %}

Unifies with the sum of elements in the list or 0 on empty list.

### 3

{% highlight prolog %}
:- pred reverse(list(A)::in, list(A)::out) is det.
{% endhighlight %}

Unifies the output list with the reverse of the input list. Hints:

-   Use a list as the accumulator.
-   Base case: the reverse of an empty list is the empty list.

### 4

{% highlight prolog %}
:- pred append(list(A)::in, list(A)::in, list(A)::out).
{% endhighlight %}

Unifies the output list to the concatenation of the first list input
list with the second one.

Hints:

-   Try to implement this without using LCO first - it will be much
    easier.
-   Base case: appending an empty list to anything yields the same list
-   After you do that - use a list as the accumulator, use reverse on
    it!

`true` and `false`
==================

To explicitly `fail` a predicate (like for example on some condition)
use `false`, likewise `true` will always succeed. This will help us
define some predicates (in particular `whitespace1`).

Parsing whitespace
==================

So, after this theoretical introduction we're now ready to extend our
program.

Goals
-----

To parse the grammar defined by the BNF above we'll introduce three new
predicates corresponding to the rules:

1.  `:- pred rule(char::out, list(char)::in, list(char)::out) is semidet.`
2.  `:- pred whitespace(list(char)::in, list(char)::out) is det.`
3.  `:- pred whitespace1(list(char)::in, list(char)::out) is semidet.`

Testing
-------

{% highlight prolog %}
$ ./main '    a'
No match!
$ ./main '    !'
Found match: !
$ ./main '@'
Found match: @
$ ./main '   $ '
Found match: $
{% endhighlight %}

Implementation
--------------

The full code for this part is
[here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/05)

Let's skip the obvious parts that haven't changed:

{% highlight prolog %}
:- module main.

:- interface.

:- import_module io.

:- pred main(io, io).
:- mode main(di, uo) is det.

:- implementation.

:- import_module char, string, list.

:- pred symbol(char::out, list(char)::in, list(character)::out) is semidet.
symbol(C, ListIn, ListOut) :-
    ListIn = [C | ListOut],
    string.contains_char("!$%&|*+-/:<=>?@^_~#", C).
{% endhighlight %}

So for the meat and bones let's have a look at the `whitespace1`
predicate

{% highlight prolog %}
:- pred whitespace1(list(char)::in, list(char)::out).
whitespace1(ListIn, ListOut) :-
    ListIn = [C | ListOut],
    ( C = ' ' ->
      true
    ; C = '\t').
{% endhighlight %}

So the `whitespace1` parser first tries to extract the head and tail of
the list, we can unify `ListOut` with the tail here - `whitespace1` only
consumes one char.

Then we compare the char with ' ' - in the consequent we explicitly
succeed using `true` and in the alternative we compare `C` with = '\\t'
(either failing or succeeding).

{% highlight prolog %}
:- pred whitespace(list(char)::in, list(char)::out) is semidet. 
whitespace(ListIn, ListOut) :-
  ( whitespace1(ListIn, Rest) ->
    whitespace(Rest, ListOut)
  ; ListIn = ListOut).
{% endhighlight %}

Here we have the recursive definition of whitespace - nothing fancy. The
base case is the 0-repetitions of whitespace where we succeed unifying
`ListIn` to `ListOut` - and for the recursive step we unify the `Rest`
of the characters with whitespace.

{% highlight prolog %}
:- pred rule(char::out, list(char)::in, list(character)::out) is semidet.
rule(Symbol, ListIn, ListOut) :-
  whitespace(ListIn, Without_Whitespace),
  symbol(Symbol, Without_Whitespace, ListOut).
{% endhighlight %}

Here we just pass the input list to the `whitespace` parser possibly
stripping out any optional whitespace and then passing **that** to the
`symbol` parser.

The updated main `predicate` looks like this:

{% highlight prolog %}
main(IO_1, IO_Last) :-
    io.command_line_arguments(Arguments, IO_1, IO_2),
    ( Arguments = [First | _Rest] ->
      string.to_char_list(First, CharList),
      ( rule(C, CharList, _Other_Chars) ->
        io.write_string("Found match: " ++ char_to_string(C) ++ "\n", IO_2, IO_Last)
      ; io.write_string("No match!\n", IO_2, IO_Last))
    ; io.write_string("No arguments given!\n", IO_2, IO_Last)).
{% endhighlight %}

Ending words
============

Uff! That's all for this part - in the next part we'll define the core
data structures for our interpreter, then we'll have sidetrack a bit and
discuss better parsing methods.

[Next](06-algebraic-data-types-and-the-base-of-our-interpreter.html)
