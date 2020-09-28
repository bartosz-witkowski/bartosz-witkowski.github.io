---
layout: wyas
title: DCGs and Parsing Continued
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

DCGs and Parsing Continued 
==========================

In this part we will add parsers to the data types that we created
previously, also we'll learn about some syntax sugar that makes writing
parsers somewhat easier. Let's get started!

New concepts 
============

New predicates 
--------------

Unifies if the input char is matches "a-zA-Z"

{% highlight prolog %}
:- pred char.is_alpha(char::in) is semidet.
{% endhighlight %}

Unifies if the input char is matches "a-zA-Z"

{% highlight prolog %}
:- pred char.is_digit(char::in) is semidet.
{% endhighlight %}

Tries to parse the string as an integer value. Fails if not possible.

{% highlight prolog %}
:- func integer.from_string(string::in) = (integer::out) is semidet.
{% endhighlight %}

Converts a list of characters to a `string`. Note the `uo` mode in the
output argument and see the next section.

{% highlight prolog %}
:- pred string.from_char_list(list(char)::in, string::uo).
:- func string.from_char_list(list(char)::in) = (string::uo) is det.
{% endhighlight %}

More on uniqueness 
------------------

In the previous parts we said that `uo` was `free >> unique` and that
`out` modes were `free` » `ground`. We *can* convert a variable that's
`unique` to `ground` - but that `unique` variable will cease to be
unique.

Now you might wonder "why"?

1.  Won't this destroy reasoning about effects?
2.  Why use `uo` anywhere but in effectful predicates?

For 1. - no! If you think about it you'll see that in the `main`
predicate expects a pair of `(di, uo)` arguments - and doing anything to
an `unique` variable will clobber it. The mode system would not allow
anything done to the `io` type that would not preserve uniquness.

As for 2 - for unique variables mercury is able to do some magic
optimizations - such as destructive updates (if we know that a string is
unique we can change it *in place* and in the end the output won't
matter).

But don't think too strongly about point 2 for now :).

DCGs
----

While writing a parser using "just" plain old predicates is certainly
possible we'll introduce DCGs which will help ease the pain of writing
out parsers.

We won't go much into the theory of DCGs - I recommend reading [the
wikipedia entry](http://en.wikipedia.org/wiki/Definite_clause_grammar)
and [this prolog tutorial about
DCGs](http://www.pathwayslms.com/swipltuts/dcg/). We also won't cover
"advanced" usages of DCGs such as passing state.

You may have noticed that the parser predicates that we wrote follow a
certain pattern:

1.  Predicates in which we want the output argument (match) have that
    argument first, followed by the input list and the output list.
2.  Predicates that have no output argument have only the input list
    followed by the output list.

This was done explicitly for the purpose of compatibility with DCGs.

We will view DCGs as a simple syntax transformation (syntax sugar) that
can transform predicates in the form:

{% highlight prolog %}
foo(t1, ..., tn) --> body.
{% endhighlight %}

to the explicit forms we have writen above (t1, ..., tn - is a list of
arguments "not counting" the input/output lists - can be empty).

In the body of a DCG rule:

1.  predicates are read as DCG predicates i.e the input/output list are
    threaded implicitly i.e

{% highlight prolog %}
foo --> bar, baz. 
{% endhighlight %}

is equivalent to:

{% highlight prolog %}
foo(List_1, List_Out) :- bar(List_1, List_2), baz(List_2, List_Out). 
{% endhighlight %}

1.  rules surrounded by a `{` and `}` will be compiled down to "normal"
    mercury predicates (so **without** implicit passing of input/output
    lists).
2.  "constant" lists are equivalent to testing the elements of the input
    list against them i.e

{% highlight prolog %}
foo ---> ['a', 'b', 'c', 'd']. 
{% endhighlight %}

is equivalent to:

{% highlight prolog %}
   foo(List_In, List_Out) :-
     List_In = ['a', 'b', 'c', 'd' | List_Out].
     % If the above is puzzling it just tries to unify
     % head of List_In with 'a', the head of the tail with 'b', etc 
     % and the rest (the tail^4) is unified with List_Out 
{% endhighlight %}

Why is this useful? Let's tackle an example of a BNF grammar from [this
page](http://www.cis.upenn.edu/~matuszek/cit594-2013/Assignments/05a-bnf-example.html)
(simplified to remove repetitions '{' X '}' with recursion)

{% highlight prolog %}
<program> ::= <reads> <writes>.
<reads> ::= "read" <var> <reads> | "".
<writes> ::= "write" <write_list>
<write_list> ::= <var> "," <write_list> | <var>.
<var> ::= a | b | c | d | e.
{% endhighlight %}

and here's how this would look like in mercury using DCG-syntax ([full
code
here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/07/example))

{% highlight prolog %}
:- pred program(list(char)::in, list(char)::out) is semidet.
program --> reads, writes.

:- pred reads(list(char)::in, list(char)::out) is det.
reads --> (['r', 'e', 'a', 'd'], var -> reads ; []).

:- pred writes(list(char)::in, list(char)::out) is semidet.
writes --> ( ['w', 'r', 'i', 't', 'e'], write_list -> writes ; [] ).

:- pred write_list(list(char)::in, list(char)::out) is semidet.
write_list --> (var,  [','] -> write_list ; var).

:- pred var(list(char)::in, list(char)::out) is semidet.
var --> ['a'] ; ['b'] ; ['c'] ; ['d'] ; ['e'].
{% endhighlight %}

Hopefully it's easy to see the similarity between DCGs and the BNF
grammar - without having to explicit pass the input and output char
lists it's much much clearer to see the "what" and not the "how".

Quiz
----

Rewrite the following DCGs into predicate form, add signatures where
non-trivial. Write the BNF grammar that corresponds to the DCG.

### 1. 

{% highlight prolog %}
dcg1 --> ['!'].
{% endhighlight %}

### 2. 

{% highlight prolog %}
dcg2(X) --> ['y', 'e', 's'], { X = yes }.
{% endhighlight %}

### 3. 

{% highlight prolog %}
dcg3(A) --> ( dcg2(X), dcg1 -> A = X
            ; A = no).
{% endhighlight %}

### 4. 

{% highlight prolog %}
dcg4 --> ( ['!'] -> dcg4 ; [] )
{% endhighlight %}

Rewrite the following predicates into DCG form, add signatures where
non-trivial. Write the BNF grammar that corresponds to the DCG.

### 5. 

{% highlight prolog %}
dcg5(List_In, List_Out) :-
  List_In = ['d', 'o', 'g' | List_Out].
{% endhighlight %}

### 6. 

{% highlight prolog %}
dcg6(X, List_In, List_Out) :-
  List_In = [A | List_Out],
  char.is_digit(A).
{% endhighlight %}

### 7. 

{% highlight prolog %}
dcg7(List_In, List_Out) :-
  List_In = [A | Rest],
  ( A = '-' -> 
    dcg7(Rest, List_Out)
  ; List_Out = Rest).
{% endhighlight %}

### 8. 

{% highlight prolog %}
dcg8(X, Y, List_In, List_Out) :-
  foo(X, List_In, L_1),
  bar(Y, L_1, List_Out).
{% endhighlight %}

Goal
====

The goal of this part is to parse this grammar:

{% highlight prolog %}
<atom> ::= <letter-or-symbol> <atom-rest>
<string> ::= '"' <string_content> '"'
<number> ::= <digit> | <digit> <number>
<list> ::= "" | <expression> | <expression> <space> <list>
<dotted-list> ::=  <list> '.' <space> <expression>
<quoted> = "'" <expression>

<expression> ::= <atom>
               | <string>
               | <number>
               | <quoted>
               | '(' ( <list> | <dotted-list> ) ')'

<top-level-expression> ::= <optional_space> <expression> <optional_space>

<letter> ::= regex("a-zA-Z")
<digit> ::= regex("0-9")
<symbol> ::= '!' | '$' | '%' | '&' | '|' | '*' | '+' | '-' 
           | '/' | ':' | '<' | '=' | '>' | '?' | '@' | '^'
           | '_' | '~' | '#'

<optional_space> ::= "" | (' ' | '\t') <optional_space>
<space> ::= (' ' | '\t') <optional_space>

<letter-or-symbol> = <letter> | <symbol>
<string_content> ::= "" | regex("^\"") <string_content>
<atom-rest> ::= "" | (<letter> | <digit> | <symbol>) | <atom-rest>
{% endhighlight %}

Caveats:

1.  In recursive predicates we don't discern between a "proper" end of
    input and an improper one i.e '55a' will be parsed as "55" and the
    parser will stop on unknown char 'a'.
2.  To properly parse forms `()`, `(a)` and `(a b)` we introduce 3 rules
    in `<list>` - one for empty list, one for 1 element list and one for
    lists with spaces after the first element.

This should explain the weird looking `dotted-list` rule - intuitively
we'd expect that as:

{% highlight prolog %}
<dotted-list> ::= <list> <spaces> '.' <spaces> <expression>
{% endhighlight %}

but because `<list>` consumes spaces in last alternative and failures
will

Parsing is only the first step - we also need to produce values:

1.  `<atom>` rule's output should produce:

    -   `lisp_boolean(yes)` if "\#t" was parsed
    -   `lisp_boolean(no)` if "\#f" was parsed
    -   `lisp_atom(PARSED)` (where PARSED was not "\#t" nor "\#f")

2.  `<number>` rule's output should produce `lisp_number(PARSED)`

3.  `<quoted>` rule's output should produce
    `lisp_list([lisp_atom("quote"), PARSED_EXPRESSION])`

4.  `<list>` rule's should unify with `lisp_list(PARSED_EXPRESSIONS)`

5.  `<dotted-list>` rule's output should unify with
    `lisp_dotted_list(PARSED_EXPRESSIONS_BEFORE_THE_DOT, PARSED_EXPRESSION)`

Parsing and producing values should be done in module `parsing` only
`top_level_expression` should be exported.

We'll also need changes in the `main` module. We want to discern between
"partial" matches and full matches - we have to check the output char
list and see if it's empty after unifying with `top_level_expression`.

Tests
=====

{% highlight prolog %}
$ ./main
No arguments given!
$ ./main 5
Found match!
$ ./main a
Found match!
$ ./main 5a
Partial match!
$ ./main 'a b'
Partial match!
$ ./main '()'
Found match!
$ ./main '(a)'
Found match!
$ ./main '(a b)'
Found match!
$ ./main '(a b c)'
Found match!
$ ./main '(a b 5a)'
No match!
$ ./main '(a . b)'
Found match!
$ ./main '#t'
Found match!
$ ./main '    '
No match!
{% endhighlight %}

Tips
====

1.  Start small - implement `number` parser first and flesh out
    `top_level_expression`, `expression` and the changes in the main
    module.
2.  After that do `string`, `atom`, `quoted` and then `list` and finally
    `dotted_list` predicates. Test after each new addition.
3.  Use two types of predicates/DCGs - ones for parsing and ones for
    producing values. E.g in my code I used `parse_number` to unify with
    the digits themselves but the `number` predicate takes the output of
    `parse_number` and has one number output that unifies with an
    integer.

Implementation
==============

For the whole project you can check out [this
directory](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/07/code)
.

Let's start from the changes in the main module:

{% highlight prolog %}
:- import_module parser.

main(IO_1, IO_Last) :-
  io.command_line_arguments(Arguments, IO_1, IO_2),
  ( Arguments = [First | _Rest] ->
    string.to_char_list(First, CharList),
    ( parser.top_level_expression(_E, CharList, Rest) ->
      ( Rest = [] -> 
        io.write_string("Found match!\n", IO_2, IO_Last)
      ; io.write_string("Partial match!\n", IO_2, IO_Last))
    ; io.write_string("No match!\n", IO_2, IO_Last))
  ; io.write_string("No arguments given!\n", IO_2, IO_Last)).
{% endhighlight %}

This is getting a little convoluted but bare with me - we will learn
some syntax sugar that helps with code such as that in due time. The
code while looking scary is simple - just nested `if`s. And the only
real part that has changed is this:

{% highlight prolog %}
    ( parser.top_level_expression(_E, CharList, Rest) ->
      ( Rest = [] -> 
        io.write_string("Found match!\n", IO_2, IO_Last)
      ; io.write_string("Partial match!\n", IO_2, IO_Last))
{% endhighlight %}

So like we said before we want to find either an partial or a full match
- so we check `Rest` and match it against the empty list. If the list is
indeed empty we have a full match, if it's not - it's a partial match.

Now for the `parser` module. The header:

{% highlight prolog %}
:- module parser.

:- interface.
:- import_module char, list.
:- import_module adts.

:- pred top_level_expression(lisp_val::out, list(char)::in, list(char)::out) is semidet.
{% endhighlight %}

Nothing fancy - we export the only predicate that we will use - and it's
a parser that will unify with `lisp_val`s.

In the implementation section we need to import other modules that we
will use:

{% highlight prolog %}
:- implementation.

:- import_module bool, string, list, integer.
{% endhighlight %}

Now the definitions of `top_level_expression` and `expression`:

{% highlight prolog %}
top_level_expression(X) --> optional_space, expression(X), optional_space.

:- pred expression(lisp_val::out, list(char)::in, list(char)::out) is semidet.
expression(X) -->
  ( atom(Y)   -> { X = Y } 
  ; string(Y) -> { X = Y }
  ; number(Y) -> { X = Y }
  ; quoted(Y) -> { X = Y }
  ; ['('], 
    ( dotted_list(Y) -> { X = Y }
    ; list(X)),
    [')']).

% ...

:- pred optional_space(list(char)::in, list(char)::out) is det. 
optional_space -->
  ( [' ']  -> optional_space
  ; ['\t'] -> optional_space
  ; [] ).
{% endhighlight %}

`top_level_expression` just follows the BNF grammar, so nothing
surprising - it uses the `optional_space` predicate that uses recursion.
If we'd try to analyze it we can see that if it find an `' '` empty
space it will try to match more, same with the tab `'\t'` and if it
finds neither it stops.

`expression` on the other had just delegates to the other `parsers`.
Let's analyze them one by one starting from the easiest:

`number` 
--------

For all the parsers that produce `lisp_val`s we will also write a parser
that just matches tha characters without any additional logic. For
clarity we will prefix them with `parse_`.

The definition of `parse_number` is:

{% highlight prolog %}
:- pred parse_number(list(char)::out, list(char)::in, list(char)::out) is semidet.
parse_number(Chars) --> ( digit(X) ->
                          { Chars = [X | Xs] }, parse_number(Xs)
                        ; { Chars = [] } ).

% ...

:- pred digit(char::out, list(char)::in, list(char)::out) is semidet.
digit(X) --> [X], { char.is_digit(X) }.
{% endhighlight %}

`digit` first matches any character then uses a normal (i.e non DCG)
predicate to check if that character is in the range 0-9. So we can say
it just dcg-ifies the `digit` predicate.

`parse_number` tries to consume as many digits as possible. Recursively
we append the current digit with the rest of the digits where the empty
list is base case for recursion.

For the `lisp_val`-producing predicate:

{% highlight prolog %}
:- pred number(lisp_val::out, list(char)::in, list(char)::out) is semidet.
number(Number) --> parse_number(Cs), { 
                     string.from_char_list(Cs, String),
                     Integer = integer.from_string(String),
                     Number = lisp_number(Integer)
                   }.
{% endhighlight %}

We unify with `parse_number(Cs)` first - then obtain a string from the
character list using `string.from_char_list`, next we convert the
`string` into an `integer` using `integer.from_string` (possibly failing
here if our parser ain't right) and finally we wrap the integer with the
`lisp_number` constructor.

`string` 
--------

String is not that much different from number, first let's look at the
`parse_string` predicate:

{% highlight prolog %}
:- pred parse_string(list(char)::out, list(char)::in, list(char)::out) is semidet.
parse_string(CharList) --> 
  [ '"' ], 
  string_content(CharList), 
  [ '"' ]. 
{% endhighlight %}

`parse_string` is *really* simple just using a combination of three
other rules.

{% highlight prolog %}
:- pred string_content(list(char)::out, list(char)::in, list(char)::out) is semidet.
string_content(Chars) -->  
  ( not_quote(X) ->
    string_content(Xs), { Chars = [X | Xs] }
  ; { Chars = [] }).

:- pred not_quote(char::out, list(char)::in, list(char)::out) is semidet.
not_quote(X) --> [X], { X \= '"' }.
{% endhighlight %}

We use `not_quote` as a helper rule - nothing interesting going on in
there. `string_content` is tries to consume as many "not quotes" as
possible unifying with `Chars` with the current *not-quote-character*
cons-ed with the rest of the *not-quote-characters* in the input list.

And now `string`:

{% highlight prolog %}
:- pred string(lisp_val::out, list(char)::in, list(char)::out) is semidet.
string(String) -->
  parse_string(CharList), {
    String = lisp_string(string.from_char_list(CharList))
  }.
{% endhighlight %}

Here we just convert the char list into a string and wrap that in the
`lisp_string` constructor.

atom
----

Parsing code:

{% highlight prolog %}
:- pred parse_atom(list(char)::out, list(char)::in, list(char)::out) is semidet.
parse_atom(Name) -->
  ( letter(Y) -> 
    { X = Y }
  ; symbol(X) ),
  parse_atom_rest(Xs),
  {
    Name = [X | Xs]
  }.

% ...
:- pred letter(char::out, list(char)::in, list(char)::out) is semidet.
letter(X) --> [X], { char.is_alpha(X) }.

:- pred symbol(char::out, list(char)::in, list(char)::out) is semidet.
symbol(X) --> [X], { string.contains_char("!$%&|*+-/:<=>?@^_~#", X) }.
{% endhighlight %}

`parse_atom` will unify with the characters of the atom name, the head
must either be `letter` or `symbol` - the former is a simple predicate
and the latter we know from the previous parts.

If everything succeeded up to this point we then parse the rest of the
atom name using the `parse_atom_rest` predicate and we unify the name by
appending the first char with the output list of `parse_atom_rest`.

{% highlight prolog %}
:- pred parse_atom_rest(list(char)::out, list(char)::in, list(char)::out) is semidet.
parse_atom_rest(Rest) -->
  ( letter(X) -> parse_atom_rest(Xs), { Rest = [X | Xs] }
  ; digit(X)  -> parse_atom_rest(Xs), { Rest = [X | Xs] }
  ; symbol(X) -> parse_atom_rest(Xs), { Rest = [X | Xs] }
  ; { Rest = [] }).
{% endhighlight %}

We should be used to sights like this by now - again a recursive rule
and failing will unify the output with the empty list.

And this is all used by `atom`:

{% highlight prolog %}
:- pred atom(lisp_val::out, list(char)::in, list(char)::out) is semidet.
atom(Atom) -->
  parse_atom(Chars),
  {
    AtomName = string.from_char_list(Chars),
    ( AtomName = "#t" ->
      Atom = lisp_bool(yes)
    ; AtomName = "#f" ->
      Atom = lisp_bool(no)
    ; Atom = lisp_atom(AtomName))
  }.
{% endhighlight %}

First we use the `parse_atom` rule then we take the unified chars and
covert them into a string - and we match on that. Either
`lisp_bool(yes)`, `lisp_bool(no)` or `lisp_atom(AtomName)` will be
produced.

quoted
------

The quoted rule is really not that difficult to grasp:

{% highlight prolog %}
:- pred quoted(lisp_val::out, list(char)::in, list(char)::out) is semidet.
quoted(Quoted) --> 
        [ '\'' ],
        expression(Expression),
        {
                Quoted = lisp_list([lisp_atom("quote"), Expression])
        }.
{% endhighlight %}

It uses just uses the previously defined `expression` passes that to a
`lisp_lisp` with two elements `lisp_atom("quote")` and the second being
the expression.

Note the mutual recursion between `expression` and `quoted` - think
about this for a bit if it seems hard - but it's really no different to
"normal" recursion.

list
----

While the `parse_` predicates before didn't produce `lisp_val`s the
`parse_list` here also won't produce a single `lisp_val`- but it will
produce a list of them. We define `parse_list` so it can be reused later
on in `parse_dotted_list`.

{% highlight prolog %}
:- pred parse_list(list(lisp_val)::out, list(char)::in, list(char)::out) is det.
parse_list(List) -->
  ( expression(Expression), space, parse_list(Rest) -> { List = [ Expression | Rest ] }
  ; expression(Expression) -> { List = [Expression] }
  ; { List = [] } ).
{% endhighlight %}

Just nested `if-then-else` statements just like in the BNF grammar, we
use `expression` once again and the non optional `space` rule which
looks like this:

{% highlight prolog %}
:- pred space(list(char)::in, list(char)::out) is semidet. 
space -->
  ( [' ']  -> optional_space
  ; ['\t'], optional_space).
{% endhighlight %}

`space` tries to consume at least one `' '` or `'\t'` and then it
"delegates" to `optional_space` rule.

Finally we wrap the `list(lisp_val)` in a `lisp_list` in the `list`
predicate:

{% highlight prolog %}
:- pred list(lisp_val::out, list(char)::in, list(char)::out) is det.
list(List) --> parse_list(Xs), { List = lisp_list(Xs) }.
{% endhighlight %}

dotted\_list
------------

In `parse_dotted_list` we reuse the `parse_list` rule then match against
the dot `.`, `space` and `expression` - see the caveats section in the
goal if you don't know why it looks like that.

{% highlight prolog %}
:- pred parse_dotted_list(list(lisp_val)::out, lisp_val::out, list(char)::in, list(char)::out) is semidet.
parse_dotted_list(List, Expression) -->
  parse_list(List), ['.'], space, expression(Expression).
{% endhighlight %}

`parse_dotted_list` unifies with *two* output variables - a list and an
expression we use that in `dotted_list`...

{% highlight prolog %}
:- pred dotted_list(lisp_val::out, list(char)::in, list(char)::out) is semidet.
dotted_list(Dotted_List) -->
  parse_dotted_list(Xs, X), { Dotted_List = lisp_dotted_list(Xs, X) }.
{% endhighlight %}

... wrapping both in the `lisp_dotted_list` constructor.

End
===

In the next parts we will concentrate on writing simple evaluators for
the values that we parsed starting with converting them to strings.

[Next](08-typeclasses-and-show.html)
