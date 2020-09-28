---
layout: post
title: Polymorphism, and typeclasses in Scala.
permalink: /2013/03/29/polymorphism-and-typeclasses-in-scala.html
categories:
- blog
---

First of all, this post is long overdue. When I've been reading up on Scala,
even without going into the nitty-gritty, I've found that my knowledge was
lacking. I was full of misconceptions and mental crutches from OOP. What I knew
from the OOP-land was just a tip of the iceberg. 

This post will be mostly about typeclasses, and I'll try to summarize what I've
learned about polymorphism in Scala. The motivation for this is simple - I've
begun to ponder what would I say given a simple interview question: "Could you
explain polymorphism?". 

Previously my reaction to this would be to describe inheritance, virtual methods
and maybe going into v-tables. Now my response wold be more akin to "What kind
of polymorphism?".

There are kinds?
================

What I had grokked.
-------------------

Subtype polymorphism was the only type of polymorphism I was aware of. Subtype
polymorphism is the bread and butter of OOP. The typical introduction commonly
shows it by using analogies to real life objects. Not being original I'll use
animals performing various sounds. I know using classes to mimic real life
objects isn't a very good idea (as seen in the circle-ellipse problem), but bear
with me for the sake of example.

{% highlight scala %}
scala> trait Animal {
     |   def makeSound: String
     | }
defined trait Animal

scala> class Dog extends Animal {
     |   override def makeSound = "Woof"
     | }
defined class Dog

scala> class Cat extends Animal {
     |   override def makeSound = "Meow"
     | }
defined class Cat

scala> // Let's create values for cat and dog that we'll use in the examples:

scala> val dog = new Dog
dog: Dog = Dog@5e8dc627

scala> val cat = new Cat
cat: Cat = Cat@f74f6ef

scala> val animals = List(dog, cat)
animals: List[Animal] = List(Dog@5e8dc627, Cat@f74f6ef)

scala> for (animal <- animals) {
     |   println(animal.makeSound)
     | }
Woof
Meow
{% endhighlight %}

The gist of subtype polymorphism is that we can abstract away some computation
(returning a string) depending on what that object is. A `Cat` **is a** Animal
and can substitute for it.

The previously mentioned v-table is the usual implementation of subtype
polymorphism. A class stores a table of function pointers to the implementations
of functions. Calling the `makeSound` method on a `Cat` object would first find
the address of the `makeSound` method (i.e. it would point `Cat::makeSound`
using "C++ terminology").

Whether a particular implementation uses v-tables is beside the point, really.
The design should just allow for dynamic dispatch: what method is called depends
on the *runtime* type of the object. 

What I knew...
--------------

is generic programming. But I didn't know it was called parametric polymorphism
in the functional-programming world.

With contrast to subtype polymorphism parametric polymorphism lets us define
computations/data structures that can abstract over any type (or a subset) while
*writing it*. 

The way I think about it, a generic function/data structure doesn't serve as an
abstraction by itself it's a "template" for future use.  One cool thing about it
is that generic methods usually don't care about the particular type - it will
be specified at some future time. Generic data types take the principle of least
knowledge up to eleven.

Example time, a simple generic class: 

{% highlight scala %}
scala> case class Box[T](value: T)
defined class Box
{% endhighlight %}

The type of the Box will be defined at some point in the future:

{% highlight scala %}
scala> val box = Box(2) 
box: Box[Int] = Box(2)
{% endhighlight %}

I'll try to tackle generic programming in Scala seriously in another post. For
now, this will do.

What I did not know.
--------------------

Was ad-hoc polymorphism. Ad-hoc polymorphism allows us to abstract a computation
over some subtype like parametric polymorphism. Unlike parametric polymorphism
the type of the argument is important because it's used to dispatch to a
concrete implementation. Ad-hoc polymorphism can be though of as a mechanism for
function overloading. 

In contrast to subtype polymorphism ad-hoc polymorphism "is done" at compile
time - I'll show an example of this at the end.

Typeclasses
===========

Quoting after [wikipedia](http://en.wikipedia.org/wiki/Type_class) a typeclass
(or a type class) is a construct that supports ad-hoc polymorphism. In Haskell a
type class is a language construct, but scala can get away with using mechanisms
it's got already: implicit classes.

I've written something about implicits
[here](/2012/07/30/ordering-and-ordered-in-scala.html). You can check it out, as
a means of introduction.

So how can we implement type classes in Scala? 

Lets forget about implicits for a sec, and get back to the animal example.
Suppose we wanted to add a method that returns the name of offspring of an
animal. We could do it by providing an parametrized `offspringName` operation

{% highlight scala %}
scala> trait OffspringName[T] {
     |   def offspringName(t: T): String
     | }
defined trait OffspringName
{% endhighlight %}

Instead of mixing in the trait, we define a 'helper method' that uses the
`OffspringName` trait to give the result:

{% highlight scala %}
scala> def offspringName[T](t: T)(o: OffspringName[T]): String = {
     |   o.offspringName(t)
     | }
offspringName: [T](t: T)(o: OffspringName[T])String
{% endhighlight %}

Now we can define concrete implementations for the `OffspringName` operation

{% highlight scala %}
scala> object OffspringName {
     |   object CatHasOffspringName extends OffspringName[Cat] {
     |     override def offspringName(cat: Cat) = "Kitty"
     |   }
     |   object DogHasOffspringName extends OffspringName[Dog] {
     |     override def offspringName(dog: Dog) = "Puppy"
     |   }
     | }
defined module OffspringName
warning: previously defined trait OffspringName is not a companion to object OffspringName.
Companions must be defined together; you may wish to use :paste mode for this.

scala> // Ignore the warning it's not really important here.
{% endhighlight %}

Now this works:

{% highlight scala %}
scala> import OffspringName._
import OffspringName._

scala> offspringName(cat)(CatHasOffspringName)
res1: String = Kitty

scala> offspringName(dog)(DogHasOffspringName)
res2: String = Puppy
{% endhighlight %}

And because the types don't match this does not:

{% highlight scala %}
scala> offspringName(cat)(DogHasOffspringName)
<console>:18: error: type mismatch;
 found   : OffspringName.DogHasOffspringName.type
 required: OffspringName[Cat]
              offspringName(cat)(DogHasOffspringName)

scala> offspringName(dog)(CatHasOffspringName)
<console>:18: error: type mismatch;
 found   : OffspringName.CatHasOffspringName.type
 required: OffspringName[Dog]
              offspringName(dog)(CatHasOffspringName)
{% endhighlight %}

As can be expected the actual implementation of the type class can be provided
by an implicit argument like so:

{% highlight scala %}
// modify the "helper method"
scala> def offspringName[T](t: T)(implicit o: OffspringName[T]): String = {
     |   o.offspringName(t)
     | }
offspringName: [T](t: T)(implicit o: OffspringName[T])String

// and add implicits to the implementations
scala> object OffspringName {
     |   implicit object CatHasOffspringName extends OffspringName[Cat] {
     |     override def offspringName(cat: Cat) = "Kitty"
     |   }
     |   implicit object DogHasOffspringName extends OffspringName[Dog] {
     |     override def offspringName(dog: Dog) = "Puppy"
     |   }
     | }
defined module OffspringName
warning: previously defined trait OffspringName is not a companion to object OffspringName.
Companions must be defined together; you may wish to use :paste mode for this.

scala> import OffspringName._
import OffspringName._

scala> offspringName(cat)
res3: String = Kitty

scala> offspringName(dog)
res4: String = Puppy
{% endhighlight %}

This only works for implicits that are in scope, defining a new `Animal` class
won't magically provide an implementation

{% highlight scala %}
scala> class Cow extends Animal {
     |   override def makeSound: String = "Moo!"
     | }
defined class Cow

scala> val cow = new Cow
cow: Cow = Cow@47cb3b45

scala> offspringName(cow)

<console>:23: error: could not find implicit value for parameter o: OffspringName[Cow]
              offspringName(cow)
{% endhighlight %}

Of course we can add it as before. And that's what's **really cool** about
typeclasses. We can add new functionality to old code without modifying it. 

While in subtype polymorphism dispatch is done through the runtime type of an
object, typeclasses dispatch on the compile time type. See: 

{% highlight scala %}
scala> val catAnimal: Animal = cat
catAnimal: Animal = Cat@51927ba1

scala> cat.makeSound
res5: String = Meow

scala> catAnimal.makeSound
res6: String = Meow

scala> offspringName(cat)
res7: String = Kitty

scala> offspringName(catAnimal)
<console>:23: error: could not find implicit value for parameter o: OffspringName[Animal]
              offspringName(catAnimal)
                           ^
{% endhighlight %}


Adding sugar 
------------

Scala provides additional syntactic sugar for the implicit argument called view
bounds. As an alternative to:

{% highlight scala %}
scala> def offspringName[T](t: T)(implicit o: OffspringName[T]): String = {
     |   o.offspringName(t)
     | }
{% endhighlight %}

We can say:

{% highlight scala %}
scala> def offspringName[T : OffspringName](t: T): String = {
     |  implicitly[OffspringName[T]].apply(t)
     | }
{% endhighlight %}

As a side note the implicit parameter is called an `evidence` - we can interpret
this as is there an evidence in the implicit scope that the type T supports the
`OffspringName` "operation".

Another trick is to have an implicit conversion providing a suffix method call:

{% highlight scala %}
scala> implicit class OffspringNameOp[T : OffspringName](t: T) {
     |   def offspring: String = offspringName(t)
     | }
defined class OffspringNameOp

scala> cat.offspring
res9: String = Kitty

scala> dog.offspring
res10: String = Puppy

{% endhighlight %}

To summarize
============

* Only subtype polymorphism dispatches on runtime type
* Parametric polymorphism mostly doesn't care about the type it abstracts over.
* Subtype polymorphism is the dual of ad-hoc polymorphism: to change the behaviour 
  in subtype polymorphism you change the type of the polymorphic object; in ad-hoc
  polymorphism you change the type of the target. 
* Typeclasses in scala work through a combination of a generic traits and
  implicit conversions.
* Typeclasses provide type-safe extensions to allready defined classes.
