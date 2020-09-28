---
layout: post
title: Logic Programming in Scala Part II
subtitle: Reintroducing safety.
permalink: /2020/07/10/logic-programming-in-scala-part-2.html
categories:
- blog
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

In the [previous blog post]({% post_url blog/2020-07-07-logic-programming-in-scala %}), I introduced a logic programming system, inspired by [Oleg Kiselyov's toy kanren implementation](http://okmij.org/ftp/Scheme/misc.html#sokuza-kanren). In this blog post I'd like to address its two major issues: lack of variable scoping and the lack of type safety.

Variable scoping
================

The lack of variable scoping means we cannot safely reuse predicates that
introduce new variables. 

For example, if we would like to write an append predicate:

{% highlight scala %}
def append(l1: Universe, l2: Universe, l3: Universe): Predicate[Universe] = defpred(
  ((l1 === UNil) /\ (l2 === l3)) \/ 
  {
    val h = uvar("h")
    val t = uvar("t")
    val l3p = uvar("l3p")

    cons(h, t, l1) /\
    (cons(h, l3p, l3) /\ append(t, l2, l3p))
  }
}
{% endhighlight %}

We cannot guarantee in any way that the id's: `h`, `t`, `l3p` aren't already
bound in the current scope. 

What we need is to be able to declare and use completely fresh variables.

Previously, we defined `Predicate` as:

{% highlight scala %}
type Predicate[T] = Relation[Knowledge[T], Knowledge[T]]
{% endhighlight %}

For variable scoping to work we need to thread not only `Knowledge` between
predicate but also the current state of the world:

{% highlight scala %}
class State[A : Unifiable] private (
    private[logprog] val knowledge: Knowledge[A],
    private[logprog] val env: Env) {

  def setKnowledge(knowledge: Knowledge[A]): State[A] = {
    new State[A](knowledge, env)
  }

  def nextId: State[A] = {
    new State(knowledge, env.nextId)
  }
}

object State {
  private[logprog] def fresh[A : Unifiable] = new State[A](Knowledge.empty[A], Env.fresh)
}

class Env private (
  private[logprog] val currentId: Int) {

  def nextId: Env = new Env(currentId + 1)
}

object Env {
  private[logprog] def fresh() = new Env(0)
}
{% endhighlight %}

Now, our new definition of `Predicate` becomes:

{% highlight scala %}
type Predicate[T] = Relation[State[T], State[T]]
{% endhighlight %}

and the definition of `VarId` needs to become:

{% highlight scala %}
class VarId private (private[logprog] val id: Int)
{% endhighlight %}

New `VarId`s need to come directly from the state, so we can write a "smart
constructor":

{% highlight scala %}
def freshX[T : Unifiable, X](f: VarId => Relation[State[T], X]): Relation[State[T], X] = 
  succeed[State[T]].flatMap { s0 =>
    val id = new VarId(s0.env.currentId)
    val newState = s0.nextId

    succeed[State[T]].map { _ =>
      newState
    }.flatMap { _ => 
      f(id)
    }
  }

def fresh[T : Unifiable](f: VarId => Predicate[T]): Predicate[T] = 
  freshX(f)
{% endhighlight %}

We also need a way to "run" a predicate and get out the variable values:

{% highlight scala %}
def run[T : Unifiable](f: VarId => Predicate[T]) = {
  VarId.freshX[T, Option[T]] { varId =>
    
    f(varId).map { state =>
      state.knowledge.lookup(varId)
    }
  }
}.apply(State.fresh[T])

def run[T : Unifiable](f: (VarId, VarId) => Predicate[T]) = {
  VarId.freshX[T, Option[(T, T)]] { v1 =>
    VarId.freshX[T, Option[(T, T)]] { v2 =>
      f(v1, v2).map { state =>
        for {
          v1 <- state.knowledge.lookup(v1) 
          v2 <- state.knowledge.lookup(v2)
        } yield v1 -> v2
      }
    }
  }
}.apply(State.fresh[T])
{% endhighlight %}

`run` and `VarId.fresh` methods of higher arity would be constructed in a similar manner.

Armed with the above we can now safely define an `append` predicate:

{% highlight scala %}
def append(l1: Universe, l2: Universe, l3: Universe): Predicate[Universe] = defpred[Universe](
  ((l1 === UNil) /\ (l2 === l3)) \/ 
  VarId.fresh { (h, t, l3p) =>
    cons(UVar(h), UVar(t), l1) /\
    (cons(UVar(h), UVar(l3p), l3) /\ append(UVar(t), l2, UVar(l3p)))
  }
)
{% endhighlight %}

Tests:


{% highlight scala %}
scala>     println(
     |       Relation.run { x =>
     |         append(ulist(List(1).map(UInt)), ulist(List(2).map(UInt)), UVar(x))
     |       }.toList)
List(Some(UCons(UInt(1),UCons(UInt(2),UNil))))

scala>    println(
     |      Relation.run { x =>
     |        append(ulist(List(1).map(UInt)), UVar(x), ulist(List(1, 2).map(UInt)))
     |      }.toList)
List(Some(UCons(UInt(2),UNil)))

scala>    println(
     |      Relation.run { (x, y) =>
     |        append(UVar(x), UVar(y), ulist(List(1, 2).map(UInt)))
     |      }.toList)
List(Some((UNil,UCons(UInt(1),UCons(UInt(2),UNil)))), Some((UCons(UInt(1),UNil),UCons(UInt(2),UNil))), Some((UCons(UInt(1),UCons(UInt(2),UNil)),UNil)))
{% endhighlight %}

Reintroducing type safety
=========================

One shortcoming of a direct scheme port is that we lost type safety entirely.

{% highlight scala %}
/*
 * Succeeds when `x` is a member of `list`.
 */
def choice(x: Universe, list: Universe): Predicate[Universe] = defpred {
  list match {
    case UCons(h, t) => 
      (x === h) \/
      (choice(x, t))
    case other =>
      fail
  }
}
{% endhighlight %}

In particular there is nothing in the above snipped disallowing us to call `choice(UInt(1), UInt(2))` or `choice(ulist(UInt(1)), ulist(UInt(1)))` even though it could be inferred at compile time that these goals would fail.

For me, the essence of logic programming is the ability to interactively reason about unknowns. It would be a shame to let go of the ability to exclude some invalid forms at compile time.

In the rest of this blog post I'll try to implement mechanisms improving type safety in our unityped logic programming system.

Unification reconsidered
------------------------

We implemented our unification algorithm using a typeclass `Unifiable[T]` where `T` is some universe of types for which we can perform unification (logical reasoning), and we implemented this typeclass for an `Universe` type. 

If we take a look again at the `Unifiable` type class:

{% highlight scala %}
abstract class Unifiable[T] {
  def children(t: T): List[T]
  def varCheck(t: T): Option[VarId]
  def topLevelEquivalent(t1: T, t2: T): Boolean
  def mapChildren(t: T)(f: T => T): T
}
{% endhighlight %}

The `topLevelEquivalent` and `varCheck` methods are completely benign when it comes to type safety. It gets tricky when it comes to `children` and `mapChildren` as they smoosh possibly unrelated types together. What would a hypothetical typesafe unification method look like?

{% highlight scala %}
abstract class GenericUnifiable[T, Children <: HList] {
  def decompose(t: T): VarId \/ Children 
  def mapChildren(t: T)(f: Children => Children): T
  def topLevelEquivalent(t1: T, t2: T): Boolean
}
{% endhighlight %}

Unification would, unfortunately, demand passing around not only the types and their
children, but also the children's children (and their children... and so on).
[I tried sketching out](https://gist.github.com/bartosz-witkowski/44777575ee36827f0003ec794cfa5337)
an implementation but it quickly devolves into code that's not my idea of fun.

Other alternative approaches that I have not tried:
 
  * Unification with unsafe language features similar to [Typed Embedding of a Relational Language in OCaml by D. Kosarev and D. Boulytchev](https://arxiv.org/pdf/1805.11006.pdf) (on the JVM we could replace unsafe polymorphic comparison with reflection)
  * Unification via a type class a la [Typed Logical Variables in Haskell by K.  Claessen and P.  Ljunglöf](https://gup.ub.gu.se/file/207634)

I give up on trying to redefine unification as I had another approach in mind.

Predicates reconsidered
-----------------------

What we would like to work is something like this:

{% highlight scala %}
/*
 * Not compiling pseudocode
 */
def cons[A](car: A, cdr: List[A], list: List[A]): Predicate[A] = defpred {
  car :: cdr === list
}
{% endhighlight %}

Unfortunately, we only know how to unify terms belonging to some unifiable `U`
type. Additionally, for a type to work in predicates it would need to:

1. Be able to hold values.
2. Be able to work as a variable.

So instead of working on raw values we will provide a wrapper type for logical
values:

{% highlight scala %}
sealed abstract class LogicalValue[T]
object LogicalValue {
  case class Reference[T](varId: VarId) extends LogicalValue[T]
  case class Value[T](value: T) extends LogicalValue[T]

  def apply[T](value: T): LogicalValue[T] = Value(value)
}
{% endhighlight %}

Similarly, we can provide a wrapper for a logical list (a recursive data
type that's either an empty list, a variable, or a value and the rest of
the list):

{% highlight scala %}
sealed abstract class LogicalList[A] {
  def ::(a: A): LogicalList[A] = LogicalList.Cons(LogicalValue.Value(a), this)
  def ::(a: LogicalValue[A]): LogicalList[A] = LogicalList.Cons(a, this)
}

object LogicalList {
  case class Reference[A](varId: VarId) extends LogicalList[A]
  case class Cons[A](head: LogicalValue[A], tail: LogicalList[A]) extends LogicalList[A]
  case class Nil[A]() extends LogicalList[A]

  def apply[A](as: A*): LogicalList[A] = as.map(LogicalValue.apply[A]).foldRight(LogicalList.empty[A])(Cons.apply)
  def of[A](varId: VarId): LogicalList[A] = Reference[A](varId)

  def empty[A]: LogicalList[A] = Nil[A]()
}
{% endhighlight %}

Our `choice` predicate expressed with `LogicalValue`/`LogicalList` becomes:

{% highlight scala %}
/*
 * Not compiling pseudocode
 */
def cons[A](car: LogicalValue[A], cdr: LogicalList[A], list: LogicalList[A]): Predicate[A] = defpred {
  car :: cdr === list
}
{% endhighlight %}

But neither `LogicalValue` nor `LogicalList` is `Unifiable`. We *could* add it to
the example `Universe` type but we can do even something better - and make the
predicates that we write polymorphic.  Consider the following two type classes:

{% highlight scala %}
/*
 * Evidence that type `T` can be embeded in an universe `U`.
 */
abstract class Embed[T, U] {
  /** 
   * Embed the type `t` into a type in the `U` universe
   */
  def embed(t: T): U

  /*
   * Project the universe into a type `T`
   */
  def eject(u: U): Option[T]
}

/*
 * Evidence that type `T` can be reasoned about in an universe `U`
 */
abstract class Logical[T, U] extends Embed[T, U] {
  def varOf(varId: VarId): T
}
{% endhighlight %}

We only need some tiny bit of additional machinery to make universe-polymorphic
predicates work: a way to create variables when a `Logical` type class evidence
is available, and the ability to run the predicates.


{% highlight scala %}
/*
 * Create `Embed`dings in the universe `U`
 */
final class Embedding[U : Unifiable] private {
  /*
   * Create a predicate with a fresh variable of type `A` that can be embedded in universe `U`
   */
  def fresh[A](f: A => Predicate[U])(implicit l: Logical[A, U]): Predicate[U] = {
    VarId.fresh { varId =>
      f(l.varOf(varId))
    }
  }

  // we can create higher arities of `fresh` the same way

  /*
   * Run a predicate parametrized by a variable of type `A` in universe `U`
   */
  def run[A](f: A => Predicate[U])(implicit la: Logical[A, U]) = {
    VarId.freshX[U, Option[A]] { varId =>
      f(la.varOf(varId)).map { state =>
        state.knowledge.lookup(varId).flatMap(la.eject)
      }
    }.apply(State.fresh[U])
  }

  // we can implement higher arities of `run` the same way
}
{% endhighlight %}

A little bit of syntax sugar will help us along the way:

{% highlight scala %}
implicit class EmbedableSyntax[T](t1: T) {
  def =?=[U : Unifiable](t2: T)(implicit ev: Embed[T, U]) = {
    Relation.unify(t1.embed, t2.embed)
  }

  def embed[U : Unifiable](implicit emb: Embed[T, U]): U = {
    emb.embed(t1)
  }

  def ?[U : Unifiable](implicit emb: Embed[T, U]): LogicalValue[T] = {
    LogicalValue(t1)
  }
}
{% endhighlight %}

Now we can correctly implement the both `cons` and `append` predicates:

{% highlight scala %}
def cons[A, U : Unifiable](
  car: LogicalValue[A], 
  cdr: LogicalList[A], 
  list: LogicalList[A]
)(implicit e1: Embed[LogicalValue[A], U], e2: Embed[LogicalList[A], U]) = defpred[U](
  (car :: cdr) =?= list
)

def append[A, U : Unifiable](
  l1: LogicalList[A],
  l2: LogicalList[A],
  l3: LogicalList[A])(
    implicit e1: Logical[LogicalValue[A], U], 
             e2: Logical[LogicalList[A], U]): Predicate[U] = defpred[U](
  ((l1 =?= LogicalList.empty) /\ (l2 =?= l3)) \/
  Embedding[U].fresh[LogicalValue[A], LogicalList[A], LogicalList[A]] { (h, t, l3p) =>
    cons(h, t, l1) /\
    cons(h, l3p, l3) /\
    append(t, l2, l3p)
  }
)
{% endhighlight %}

To actually run and test it we need concrete evidence of
`Logical[LogicalValue[A], U]` (and the same for `LogicalList`) for some
unifiable type:

{% highlight scala %}
implicit val embedInt: Embed[Int, Universe] = new Embed[Int, Universe] {
  def embed(i: Int): Universe = UInt(i)
  def eject(u: Universe): Option[Int] = u match {
    case UInt(int) => Some(int)
    case _ => None
  }
}

implicit val embedString: Embed[String, Universe] = new Embed[String, Universe] {
  def embed(s: String): Universe = UString(s)
  def eject(u: Universe): Option[String] = u match {
    case UString(s) => Some(s)
    case _ => None
  }
}

implicit def logicalValue[A](implicit e: Embed[A, Universe]): Logical[LogicalValue[A], Universe] = 
  new Logical[LogicalValue[A], Universe] {
    override def embed(a: LogicalValue[A]): Universe = a match {
      case LogicalValue.Reference(varId) => UVar(varId)
      case LogicalValue.Value(value)     => e.embed(value)
    }
    override def eject(u: Universe): Option[LogicalValue[A]] = u match {
      case UVar(varId) => Some(varOf(varId))
      case other       => e.eject(other).map(LogicalValue.Value.apply)
    }

    override def varOf(varId: VarId): LogicalValue[A] = LogicalValue.Reference(varId)
  }

implicit def logicalList[A](implicit e: Embed[A, Universe]): Logical[LogicalList[A], Universe] = 
  new Logical[LogicalList[A], Universe] {
    val lval = logicalValue[A]

    override def embed(xs: LogicalList[A]): Universe = xs match {
      case LogicalList.Reference(varId) => UVar(varId)
      case LogicalList.Cons(h, t)       => UCons(lval.embed(h), embed(t))
      case LogicalList.Nil()            => UNil
    }
    
    override def eject(u: Universe): Option[LogicalList[A]] = u match {
      case UCons(h, t) => for {
        h <- lval.eject(h)
        t <- eject(t)
      } yield LogicalList.Cons(h, t)

      case UVar(varId) =>
	Some(LogicalList.Reference[A](varId))

      case UNil => 
        Some(LogicalList.empty[A])

      case _ => None
    }

    override def varOf(varId: VarId): LogicalList[A] = LogicalList.Reference(varId)
  }
{% endhighlight %}

Testing it all:

{% highlight scala %}
scala>    println(
     |      Embedding[Universe].run[LogicalList[Int]] { x =>
     |        cons(0.?, LogicalList(1, 2, 3), x)
     |      }.toList
     |    )
List(Some(LogicalList(0, 1, 2, 3)))

scala>    println(
     |      Embedding[Universe].run[LogicalValue[Int], LogicalList[Int]] { (x, y) =>
     |        cons(x, y, LogicalList(1, 2, 3))
     |      }.toList
     |    )
List(Some((1,LogicalList(2, 3))))

scala>    println(
     |      Embedding[Universe].run[LogicalList[Int]] { x =>
     |        append(LogicalList(1, 2, 3), LogicalList(4), x)
     |      }.toList
     |    )
List(Some(LogicalList(1, 2, 3, 4)))


scala>   println(
     |     Embedding[Universe].run[LogicalList[Int]] { x =>
     |       append(LogicalList(1, 2, 3), x, LogicalList(1, 2, 3, 4))
     |     }.toList
     |   )
List(Some(LogicalList(4)))

scala>    println(
     |      Embedding[Universe].run[LogicalList[Int], LogicalList[Int]] { (x, y) =>
     |        append(x, y, LogicalList(1, 2, 3, 4))
     |      }.toList
     |    )
List(Some((LogicalList(),LogicalList(1, 2, 3, 4))), Some((LogicalList(1),LogicalList(2, 3, 4))), Some((LogicalList(1, 2),LogicalList(3, 4))), Some((LogicalList(1, 2, 3),LogicalList(4))), Some((LogicalList(1, 2, 3, 4),LogicalList())))
{% endhighlight %}

Why all the bother?
===================

In this post we added two important features to the toy logic system described
previously: variable scoping and an ability to write type-safe predicates. 

Our strategy of writing type-safe wrappers for both values or logical lists
introduces some ceremony but it facilities writing predicates that are
polymorphic in the universe type. 

In the previous blog post we defined an `Unifiable` type class that allows
working with multiple concrete `Unifiable` instances. The reason for doing it is
simple: for logical programming in Scala to be practical unification cannot be a
closed class. New types can be unified as long as an user creates his own
`Unifiable` type class.

As library writers we can provide both an default `Unifiable` type (like we did
with the `Universe` type), type safe wrappers (like `LogicalValue[T]` and
`LogicalList[T]`) and predicates that work on any `Unifiable` `U`niverse type -
and they work as long as an `Embed`ding and `Logical` type classes for the
`U`niverse type are implemented.
