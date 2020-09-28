---
layout: post
title: Type Lambda in Scala
permalink: /2014/09/27/type-lambda-in-scala.html
categories:
- blog
---

When I first saw a type lambda in Scala code I thought to myself "What is this?!
Line noise?". Each encounter after the first one wasn't better and my eyes sort
of glazed over and I skipped reading them.

But after the initial shock wore off it turned out that type lambas are really
easy to understand if not exactly pleasant looking.

In particular the main idea about type lambda (i.e not the syntax) is really
easy to grasp, and we’ll take a look at that in this article.

Why type lambdas?
=================

The need for type lambadas arises when dealing with higher kinded types. 

Let's take a functor type as an example

{% highlight scala %}
trait Functor[A, F[_]] {
  def map[B](x: F[A])(f: A => B): F[B]
}
{% endhighlight %}

The `F[_]` generic parameter is a type of kind `* -> *` in simple terms a type
of kind `* -> *` is a type constructor that takes another type as a parameter
and "returns" a type.  Examples include `List`s, `Option`s etc.

Coming back to our functor we can easily define an instance for the `Option` type and see if it works:

{% highlight scala %}
implicit def OptionIsFunctor[A]: Functor[A, Option] = new Functor[A, Option] {
  override def map[B](x: Option[A])(f: A => B): Option[B] = x map f
}

implicitly[Functor[Int, Option]].map(Option(5))(_ + 1)
{% endhighlight %}

Now what happens if we try to define a functor for `Tuple2` where the map would
be mapping on the first argument? We might try something like this:

{% highlight scala %}
implicit def Tuple2IsFunctor[X, A](x: Tuple2[X, A]): Functor[A, Tuple2] = new Functor[A, Tuple2] {
  override def map[B](f: A => B): Tuple2[X, B] = (x._1, f(x._2))
}
{% endhighlight %}

But the compiler will rightfully complain that: `Tuple2 takes two type
parameters, expected: one`. What does that mean? Well `Tuple2` is of kind 
`(*, *) -> *` it's a type constructor with two type parameters that creates a
type, trying to plug in a type constructor of kind `* -> *` and expecting it to
work is not sound.

The hack
========

So what now? Consider this code: 

{% highlight scala %}
def Tuple2FunctorTest[X, A](x: Tuple2[X, A]) = {
  type Alias[A] = Tuple2[X, A]
  new Functor[A, Alias] {
    override def map[B](x: Alias[A])(f: A => B): Alias[B] = (x._1, f(x._2))
  }
}
{% endhighlight %}

We created a type alias with one parameter that acts like a one parameter type
alias and this is the essence of what the type lambda trick does (believe it or
not)! Here's how this looks:

{% highlight scala %}
implicit def Tuple2IsFunctor[X, A]: Functor[A, ({ type Alias[A] = Tuple2[X, A]})#Alias] = 
  new Functor[A, ({ type Alias[A] = Tuple2[X, A]})#Alias] {
    override def map[B](x: Tuple2[X, A])(f: A => B): Tuple2[X, B] = (x._1, f(x._2))
  }


implicitly[Functor[Int, ({ type Alias[A] = Tuple2[String, A] })#Alias]].map(("a", 1))(_ + 1)
// or
type StringTuple[A] = (String, A)
implicitly[Functor[Int, StringTuple]].map(("a", 1))(_ + 1)
{% endhighlight %}

To break this down a bit - we need to introduce a type alias and type alias can
only be declared in a trait, class, method or an object definition.

Unfortunately we cannot define that somewhere “in between” the def and the
return type but what we can do is define a structural type in the generic
parameter list.

That’s exactly what the `{ ... }` fragment does. The next part is pretty easy - we
define the type alias in the anonymous structural type and then we refer to it
with the `#` operator. And that's it! Most type lambdas aren’t so wordy and use
single letter parameter names (in particular scalaz seems to have a convention
of using Greek letters for the type alias something like 
`({ type λ[α] = Tuple2[X, A] })#λ`).

