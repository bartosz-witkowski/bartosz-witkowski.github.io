---
layout: post
title: Effects versus side effects
permalink: /2014/12/31/effects-versus-side-effects.html
categories:
- blog
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

When discussing pure programming, there's two "tricky terms" that often get
confused: **side effects** and **effects**. 

These terms aren't tricky because they're hard to grasp or not understandable,
but because they are often misused, or given different (sometimes conflicting)
meanings.

Quite often the 
[Humpty-Dumpty principle of definitions](https://www.goodreads.com/quotes/12608-when-i-use-a-word-humpty-dumpty-said-in-rather)
is used to explain them. 

I'll try my best define them as with the least amount of hand waving here.

Functional Programming
======================

In most of article I'll restrict myself to a subset of pure programming -
functional programming. Functional is programming is programming with functions
[1](http://en.wikipedia.org/wiki/Function_%28mathematics%29) 
[2](http://en.wikipedia.org/wiki/Captain_Obvious).

Functions are "constructs" (relations) that map the domain (we often refer to it
as input) into the co-domain (output) so that there's **exactly one** such
mapping for each input. 

In other words a function cannot return two different values for the same
inputs, or more formally:

$$
   \forall x_1, x_2 \in X \quad x_1 = x_2 \Rightarrow f(x_1) = f(x_2)
$$

where $$X$$ is the domain, and $$f(x)$$ is in the co-domain.

Equational Reasoning
====================

The term **equational reasoning** is a property that functional programs
exhibit.

In context of functional programming equational reasoning means that a
functional program can **always** (this is important) be understood as a series
of substitution steps. In other word we may **always** substitute a function
call with it's output. 

Equational reasoning is a **consequence** of functional programming, it's not a
property which can be given or taken away from functional programming.

Side effects
============

A side effect is everything that breaks equational reasoning. 

It follows that there's no such thing as a side effecting function, because a
function can only have one mapping from the domain to the co-domain and
side-effects introduce an additional "non-functional" element.

So, if side-effects are not possible, how do we read files, print to screen and
launch rockets in a functional program?

Effects
=======

Let's imagine a function that returns an integer:

{% highlight scala %}
def foo: Int = 5
{% endhighlight %}

Now, imagine that the result of this function were interpreted by a special
interpreter which printed that value out to screen (or launched $$n$$ number of
rockets, etc.). 

It's hard to say that a **function** that returns an `Int` is somehow side
effecting, even when run by the imaginary interpreter above.

Believe it or not, that's the entire secret about "printing to screen" in a
functional setting - it's not really about "special values," but a question of
"special interpreters".

To give a layman definition of an **effect**, I'd like to propose this: an effect
is a value that needs a special interpreter (usually in the form of a compiler
library) that, when interpreted, will directly affect state, values, or the
"physical world".

In Haskell there's an `IO` type which, while not special in itself, has a
special status in the Haskell compiler-library - it's the result of the `main`
function which has an `IO` type which is "later" interpreted, and the interpreter
will print to the screen, read files, launch rockets, etc.

The `main` function in Haskell is also nothing special. It's just an convention
for the compiler-library for the name of the main entry point of a program -
similarly to C's `int main(char**)` procedure or Java's `public static void
main(String[])` static method.

Because the IO type is a value like any other, a function can return it,
manipulate it, drop it, etc. It has no special meaning outside of the interpreter
run by invisibly at the end of our Haskell program.

Effects in Scala
================

While the Scala library and the compiler doesn't throw us a bone here, with some
diligence we, can also track IO effects in Scala using `scalaz.effect`

{% highlight scala %}
def program: IO[Unit] = for {
  _ <- putStrLn("Look ma, no side-effects!")
} yield ()
{% endhighlight %}


To beat the point in, there's nothing special about the `program` function or
its output value. It's just an IO type.

Unfortunately, there's also nothing special about Scala's ability to facilitate
safe programming, so we have to provide our own interpreter to interpret this
value. We do this by calling `unsafePerformIO` on the `IO` type.

{% highlight scala %}
object Main {
  def main(String[] args): Unit = {
    program.unsafePerformIO
  }
}
{% endhighlight %}

Fortunately, if we call `unsafePerformIO` exactly once at the "last possible
moment" this is relatively safe and though we cheat, we won't harm equational
reasoning this way.

Scalaz also provides an 
[SafeAppt](http://docs.typelevel.org/api/scalaz/nightly/index.html#scalaz.effect.SafeApp)
trait which can perform the above step for you.

Effects need not be monads
==========================

There's also nothing special about monads itself. While I glossed over this fact
previously, both the `IO` type in Haskell and the `IO` type provided by Scalaz
are monads. 

It's a frequent misunderstanding, but the fact that the IO types are monads is
really unimportant, and it's just implementation detail. 

To contrast this I'll show how IO looks in Mercury:

{% highlight prolog %}
:- module hello_world.

:- interface.

:- import_module io.

:- pred main(io, io).
:- mode main(di, uo) is det.

:- implementation.

main(IO_0, IO_Out) :-
	io.write_string("Hello", IO_0, IO_1),
	io.write_string(" ", IO_1, IO_2),
	io.write_string("world!", IO_2, IO_3),
	io.write_string("\n", IO_3, IO_Out)
{% endhighlight %}

The operational semantics of Mercury programs (which have more common with logic
programs than functional programs) are so alien for the functional programmer
I'll try to use functional analogues to explaining how the above works.

The `main` predicate is the analogue of the `main` function in Haskell. 

We will read the predicate `main(io::di, io::uo) is det` as
"deterministic" predicate main with *destructive input* variable of the type io,
and *unique output* variable of the type `io` as a function `io -> io`. 

There's additional trickery here that ensures that after "reading" from the
input variable once, we cannot "read" from it again. We get this trickery from
Mercury's type system which allows uniqueness typing (similar types are
available in
[Clean](http://en.wikipedia.org/wiki/Clean_%28programming_language%29)).

This "trickery" guarantees that the `io` values are built in sequence and the same
guarantees are ensured by making the `IO` type a monad.

The above program can be understood as a series of substitutions (in logical
programs, this process is called unification) and could be described as:

1. unify `IO_1` to the `io` type which when interpreted will print the string "Hello"
1. unify `IO_2` to the `io` type which when interpreted will print the string "Hello "
1. unify `IO_3` to the `io` type which when interpreted will print the string "Hello world!"
1. unify `IO_Out` to the `io` type which when interpreted will print the string "Hello world!\n"

This is a contrived example but shows how at the basics, controlling effects
isn't anything specific to a monad.
