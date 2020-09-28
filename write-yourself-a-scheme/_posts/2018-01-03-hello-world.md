---
layout: wyas
title: Hello World
permalink: /:categories/:day-:title:output_ext
categories:
- write-yourself-a-scheme
---

Installing mercury
==================

You can download mercury from the [mercury downloads page](http://dl.mercurylang.org/index.html).

Installation is pretty simple:

-   if you're on linux I'll describe the installation step by step
    below.
-   if you're on windows either download the binary release or download
    the source archive and read the installation instructions.
-   if you're on osx - read the installation instructions that come with
    the source release - they shouldn't differ much from the linux
    instructions below, though.

Installation instructions for linux {#installation_instructions_for_linux}
-----------------------------------

1.  Download the latest stable version archive from the [mercury
    download page](http://dl.mercurylang.org/index.html) in my case it
    was 14.01
2.  Extract the tar archive wherever you want it - I put it in
    `/usr/local/src`
3.  Run: `$ ./configure` - here you can also enable or disable grades
    (target platforms), but let's go with the defaults for now - it will
    be enough for this tutorial.
4.  Run: `$ make` unfortunately this will take a **long** time - so if
    you have multiple cores available instead run: `$ make -j2` (or 2,
    or 4, 6, 8 - depending on the number of cores you have)
5.  Run: `$ sudo make install` again this can take quite a while so
    running it with `-jN` is quite a good idea.

fter installing {#after_installing}
----------------

If everything went alright you should be able to call:

    $ mmc --version

On my system this prints out:

    Mercury Compiler, version 14.01, configured for x86_64-unknown-linux-gnu
    Copyright (C) 1993-2014 The University of Melbourne

If you've got any errors then something went wrong - check the outputs
of configure and/or make.

Destructive Input and Unique Output modes 
=========================================

Before we can discuss hello world there's one more thing we need to
discuss. In the previous article we talked about the `in` and `out`
modes that a predicate can have. Now we'll talk about `di`
(**D**estructive **I**nput) and `uo` (**U**nique **O**utput) modes.

Modes are defined in mercury as relations between instatnation states.
As of now we known about two instatnation states: **free** and
**ground**.

An `in` mode declares that this predicate expects a **free** variable
before unification and that variable will be **ground** after. We can
succinctly write out all modes that are of interest for now[^1]

1.  `:- mode in == ground >> ground`
2.  `:- mode out == free >> ground`
3.  `:- mode di == unique >> clobbered`
4.  `:- mode uo == free >> unique`

We see two unknown instantiation states here: **unique** and
**clobbered**. The two are duals to each other in the same way that
**free** and **ground** are. We can think of the **di** mode the same
way we think of the **in** mode with one very important distinction:
when you unify a **unique** variable it becomes **clobbered**.
**clobbered** is an instantation state that *cannot* be unified again -
trying to unify a **clobbered** variable will result in a compile-time
error.

This may seem a little strange but we'll see what this is for shortly.

[^1]: this is actually straight out of the documentation and the
      syntax `:- mode X == Y.` is how mercury defines its modes. We can
      take a look at `builtin.m` or its online documentation: [documentation](http://www.mercurylang.org/information/doc-latest/mercury_library/builtin.html) to see all the modes in their glory.  

Effects and side-effects 
========================

Effects and side-effects are really hard to define. I've tried [tackling this subject in an article](http://like-a-boss.net/2013/03/29/polymorphism-and-typeclasses-in-scala.html), but here I will try to appeal to the intuition.

A **side-effect** is everything that breaks equation reasoning i.e the program cannot be interpreted as a series of substitutions anymore.

While we can program imperative languages so that they don't break equational reasoning (i.e don't introduce side effects) the ability to introduce side effects is inherent to all imperative languages. Side effects may include:

-   Printing out to a screen (or more concretely the standard output).
-   Reading from the standard input.
-   Saving a file on a file system.
-   Reading a file from a file system.
-   Launching nuclear missiles
-   Writing out to a global variable.

Defining **effects** is trickier - while we might say that an effect is
an controlled effect I'd like to propose another (layman) definition: an
effect is a value that needs a special interpreter (usually in the form
of a compiler library) that when interpreted will directly affect state,
values or the "physical world".

This definition will come in handy soon. I promise.

Mercury, like haskell doesn't allow side-effects. As you might have
guessed by the previous section controlling effects in mercury is done
through the **di** and **uo** modes.

To start we will be interested in only one effect: printing to the
screen. For an example let's proceed to the hello world program.

Hello world 
===========

A minimal program in mercury has some ceremony behind it. This is hello
world:

{% highlight prolog %}
:- module hello_world.

:- interface.

:- import_module io.

:- pred main(io, io).
:- mode main(di, uo) is det.

:- implementation.

main(Io_In, Io_Out) :-
    io.write_string("Hello world!\n", Io_In, Io_Out).
{% endhighlight %}

Btw you can find the full code
[here](https://github.com/bartosz-witkowski/write-yourself-a-scheme/tree/master/03)

To compile it we simply run:

    $ mmc hello_world

And then run

    $ ./hello_world

If all is well you should have successfully said hello to the world. Now
let's analyze the program.

Mercury code is located inside modules. A module must have a name. The
line:

{% highlight prolog %}
:- module hello_world.
{% endhighlight %}

Declares the `hello_world` module. Next line:

{% highlight prolog %}
:- interface.
{% endhighlight %}

Declares the start of the interface section.

In the interface section we import the `io` module - we need that for
the `io` type (used by the `main` predicate).

1.  A library module needs to export all visible predicates
2.  A mercury program needs to export the "special" `main` predicate.
    `main` in mercury is special the same way that `main` in C is
    special. It's the default method that mercury tries to call (or
    rather expects in a executable).

The next lines do just that - declare the `main` predicate and its mode
and end the interface section (by starting the implementation section).

{% highlight prolog %}
:- pred main(io, io).
:- mode main(di, uo) is det.

:- implementation.
{% endhighlight %}

Finally we get the implementation of main:

{% highlight prolog %}
main(Io_In, Io_Out) :-
    io.write_string("Hello world!\n", Io_In, Io_Out).
{% endhighlight %}

The effect of `io.write_string` is writing a string to the output
stream.

IO and equational reasoning 
===========================

The way we can think of io and effects is that the `io` type (used in
the main predicate) accumulates the "changes" (or effects) that need to
be done. Then after `main` unifies the resulting `io` type is
interpreted (by some abstract mercury runtime).

In our `hello_world` program the resulting `io` type only "contains" one
io action: printing out "Hello world!\\n" - but in general it can have
much more (reading from stdin, creating files, reading files etc).

What's important that this is may not be how real mercury programs will
execute but because it the end this will be exactly the same as if they
**would** we can think of them in this way - and that's very powerful
because it preserves equational reasoning of programs.

What's up next
==============

In the next part we will actually start implementing the first parts of
our scheme interpreter. Stay tuned!

[Next](04-a-simple-parser.html)
