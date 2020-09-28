---
layout: post
title: Using the right type
permalink: /2014/05/11/using-the-right-type.html
categories:
- blog
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

*"Bad programmers worry about the code. Good programmers worry about data
structures and their relationships."*

The above quote from Linus Torvalds made **so** much sense for me the first time I
read it. Getting the key data structures right makes the rest of the task flow
naturally.

While this quote still makes sense to me, recently I have begun to appreciate
how using a good type makes everything much easier both by virtue of better
reasoning and better documentation.

Note - this blog entry is language agnostic but the examples will be in scala.

Being honest
============

How often do you see something like:

{% highlight scala %}
case class Person(name: String, surname: String, age: Int)
{% endhighlight %}

Let's assume that people always have a name and surname - which may not be true in real life [see](http://www.w3.org/International/questions/qa-personal-names). There's really not much to that class and the code seems innocuous.

But that class is a horrible lie. Do these instances make sense?
{% highlight scala %}
Person("Jane", "", 18)
Person("Jonh", "Doe", -1)
{% endhighlight %}

What does it mean for a person to have a name `""` or an empty surname? What is the meaning of a negative age?

If something **cannot** happen we should use a type that disallows it. 

Taking the `name` field as an example I can think of two ways for a type to
enforce a precondition (in this case "there are no names that are empty
strings").

1) Create a data structure that makes lying impossible by construction e.g

{% highlight scala %}
case class NonEmptyString(firstChar: Char, rest: String)
{% endhighlight %}

2) Create a data structure with a smart "partial" constructor that doesn't allow
lying.

{% highlight scala %}
class NonEmptyString private (val string: String)
object NonEmptyString {
  def apply(string: String): Option[NonEmptyString] = {
    if (strig.isEmpty) None
    else               Some(NonEmptyString(string))
  }
}
{% endhighlight %}

"Traditionally", the second example might be done with assertions 
(`require (!string.empty)`) but throwing exceptions makes the code hard to
reason about. 

A better `Person` class could look like this:

{% highlight scala %}
case class Person(name: NonEmptyString, surname: NonEmptyString, age: UInt)
{% endhighlight %}

While impossible instances of the  `Person` now cannot be constructed this
class is still not perfect. Does it makes sense for some person to be
`UInt.MaxValue` years old?  2000? 500? Does it make sense to limit the age to
200? We can add further limitations  to Age to make sure that we operate on data
that is not only possible but also makes sense.

Thinking about what values should be allowed in a data type makes one aware that
commonly we ignore the "bad apples" and nonsensical cases. While it is an
extereme position I think that in most cases using primitive data types like
`Int`, `Double`, `String`, `Long` etc should not be seen in domain specific data
types.

Keeping count.
==============

We say that some type $$A$$ has $$n$$ inhabitants - concrete values that a varible
of that type may have. In scala the boolean type has two inhabitants `true` and
`false`.  `Long`s have $$2^{64}$$ values. For a more in-depth article see this: [The
algebra of algebraic data types](http://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/)

Keeping count of the inhabitants of the type and using as much inhabitants as
needed makes the usage clear. 

The java [Comparable](http://docs.oracle.com/javase/7/docs/api/java/lang/Comparable.html)
interface is uniquely horrible. Why does an operation that only has three values
- `a > b`, `a = b`, `a < b` returns a type with $$2^{32}$$ inhabitants?  Why does
`a < b` have 2147483648 possible values? Having more inhabitants then there are
possibilities detracts from the real meaning. 

Other common deficiency is using an `Int` to pass a `Char` and signaling -1 by end of
file. Why not use:
{% highlight scala %}
sealed abstract class Read
case class ReadChar(c: Char) extends Read
object Eof extends Read
{% endhighlight %}

The meaning is clear and we don't have to worry about *millions* of unused values.

Another place where it's worth to count the inhabitants are data types with
optional values. Keeping to the `Person` example  I've often seen code that
looks like:

{% highlight scala %}
case class Person(
   name: NonEmptyString,
   surname: NonEmptyString,
   age: UInt,
   maleSpecificData: Option[MaleSpecificData],
   femaleSpecificData: Option[FemaleSpecificData])
{% endhighlight %}

Looking only at `maleSpecificData` and `femaleSpecificData` fields we can count 4
variants `(None, None), 
(None, Some(y)),
(Some(x), None),
(Some(x), Some(y))`. 

The way I've seen classes like these used the option fields were a lie and the intent was to encode
either:

{% highlight scala %}
// A person always has genderSpecificData  
case class Person(
   name: NonEmptyString,
   surname: NonEmptyString,
   age: UInt,
   genderSpecificData: Either[MaleSpecificData, FemaleSpecificData])
{% endhighlight %}

or

{% highlight scala %}
// A person may have genderSpecificData  
case class Person(
   name: NonEmptyString,
   surname: NonEmptyString,
   age: UInt,
   genderSpecificData: Option[Either[MaleSpecificData, FemaleSpecificData]],
{% endhighlight %}

Thinking about the possible inhabitants makes "lying" much harder.

Holding a promise.
================== 

Sometimes we need to hold a more specific precondition then the general data
structure allows. For example let's think about an ordered list. 

I've seen that precondition satisfied by meticulously running `.sorted` on a
list which makes the precondition distributed throughout the codebase (kind of
like the reverse of encapsulation).

While there is no `OrderedList` implementation in the scala library but we can
easily create our own either through an intermediate class or through
[Tagging](http://eed3si9n.com/learning-scalaz/Tagged+type.html).

By creating a new class like:

{% highlight scala %}
class OrderedList[T](list: List[T])

object OrderedList {
  // scalaz.Order 
  def apply(list: T)(implicit ev: Order[T)) = ???
}
{% endhighlight %}

we can encapsulate the logic and easily keep the predicate satisfied
**everywhere**. 

An ordered list is an obvious candidate for a new data structure because it is
common enough to be used and reused again - but it is worth creating a new type
even when the logic for is a one time thing. 

Putting the logic directly into a new type (even if there would be one usage of
that data type) keeps us honest.

Types as documentation
======================

Keeping rigor when creating a type makes it *very* obvious what values should
and should not go into a data type. Using a `NonEmptyString` or `NonEmptyList`
type is much better then documenting that through a comment `// the list cannot be
empty` - the former is checked by the compiler while the latter is only wishful
thinking.

Creating a "one off" data types like:
{% highlight scala %}
case class Name(chars: NonEmptyString)
{% endhighlight %}
also helps documentation and helps type safety. Why use:

{% highlight scala %}
case class Packet(crc: Int, offset: Int, // ...)
{% endhighlight %}

when this is possible:
{% highlight scala %}
case class Crc(int: Int)
case class Offset(int: Int)
case class Packet(crc: Crc, offset: Offset, // ...)
{% endhighlight %}

And while both make mistakes possible:
{% highlight scala %}
// vanilla 
val packet = Packet(offset, crc)

// with types 
val packet = Packet(Crc(offset), Offset(crc))
{% endhighlight %}

the latter makes the mistake more obvious with a good choice of variable naming.
