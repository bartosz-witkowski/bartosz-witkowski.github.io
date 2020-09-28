---
layout: post
title: Logic Programming in Scala
permalink: /2020/07/07/logic-programming-in-scala.html
categories:
- blog
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

Recently, I've found myself thinking about logic programming more and more and I wanted to try implementing a logic programming system in Scala. 

I've based my implementation on [Oleg Kiselyov's toy kanren implementation](http://okmij.org/ftp/Scheme/misc.html#sokuza-kanren),
stealing some ideas from [P. Jansson & J. Jeuring, Functional Pearl: Polytypic Unification](http://www.cse.chalmers.se/~patrikj/poly/unify/).

The essence of logic programming is twofold: relations and unification.

Relations
=========

A relation $$R$$ over sets $$X, Y$$ is a subset of the Cartesian product 
$$X \times Y$$, written $$x R y$$ such that $$x \in X, y \in Y$$. 

We will express relations in Scala using functions assigning a (possibly empty)
list of values, to values:

{% highlight scala %}
class Relation[X, Y] private (private[Relation] val f: X => LazyList[Y])
{% endhighlight %}

We use `LazyList` as the computations might be potentially infinite.

We can define two primitive relations, `fail` which associates to no result for
any value and `succeed` - a trivial relation that associates the given value to
the given value.

{% highlight scala %}
object Relation {
  def fail[A] = new Relation[A, A](_ => LazyList.empty)
  def succeed[A] = new Relation[A, A](a => LazyList(a))
}
{% endhighlight %}

Combinators
-----------

We will use combinators to create more complicated relations:

{% highlight scala %}
class Relation[X, Y] private (private[Relation] val f: X => LazyList[Y]) {
  def apply(x: X) = f(x)

  def map[Z](g: Y => Z): Relation[X, Z] = new Relation( x =>
    f(x).map(g)
  )

{% endhighlight %}

Mapping a relation $$xRy \subseteq X \times Y$$ by `g`, creates a new relation $$S \subseteq X \times Z$$ which "relates" the values of the codomain by $$g$$ - $$S = \{ (x, z) \mid x \in X, y \in Y, z \in Z \wedge xRy \wedge z = g(y) \} $$

We can test it:

{% highlight scala %}
scala> Relation.succeed[Int].map(_ + 1).apply(0).toList
val res0: List[Int] = List(1)

scala> Relation.succeed[Int].map(_ + 1).apply(2).toList
val res1: List[Int] = List(3)

scala> Relation.succeed[Int].map { x => x *  x }.apply(2).toList
val res2: List[Int] = List(4)
{% endhighlight %}


We can implement `flatMap` in similar vain - simply by `flatMap`ping on the `LazyList`:

{% highlight scala %}
def flatMap[Z](g: Y => Relation[Y, Z]): Relation[X, Z] = new Relation( x =>
  this.f(x).flatMap { y =>
    g(y).apply(y)
  }
)
{% endhighlight %}

Another combinator `disjunction`, or `\/` composes relations so that all results of both relations are returned:

{% highlight scala %}
def disjunction(other: Relation[X, Y]): Relation[X, Y] = new Relation[X, Y]( x =>
  this.f(x) #::: other.f(x)
)

def \/(other: Relation[X, Y]) = disjunction(other)
{% endhighlight %}
Test:
{% highlight scala %}
scala>   val square = Relation.succeed[Int].map { x => x * x }
val square: logprog.Relation[Int,Int] = logprog.Relation@46ba6f25

scala>    val cube   = Relation.succeed[Int].map { x => x * x * x }
val cube: logprog.Relation[Int,Int] = logprog.Relation@275bad01

scala>    val r = square \/ cube
val r: logprog.Relation[Int,Int] = logprog.Relation@17a1b96d

scala>     r(1).toList
val res0: List[Int] = List(1, 1)

scala>     r(2).toList
val res1: List[Int] = List(4, 8)

scala>     r(3).toList
val res2: List[Int] = List(9, 27)
{% endhighlight %}

Similarly, we can create a conjunction of relations: a new relation that
composes values belonging to both relations.

{% highlight scala %}
def conjunction[Z](other: Relation[Y, Z]): Relation[X, Z] = new Relation[X, Z]( x =>
  this.f(x).flatMap(other.f)
)

def /\[Z](other: Relation[Y, Z]) = conjunction(other)
{% endhighlight %}

Test:
{% highlight scala %}
scala>    val positive = Relation.succeed[Int].flatMap { x =>
     |      if (x <= 0) {
     |        Relation.fail 
     |      } else {
     |        Relation.succeed
     |      }
     |    }
val positive: logprog.Relation[Int,Int] = logprog.Relation@2fd9ec38

scala>   (r /\ positive)(-1).toList
val res3: List[Int] = List(1)

scala>    (r /\ positive)(2).toList
val res4: List[Int] = List(4, 8)

scala>    (r /\ positive)(-3).toList
val res5: List[Int] = List(9)
{% endhighlight %}

Logic Variables and Unification
===============================

Taking a page out of Oleg Kiselyov's description - a logic variable is a way of encoding improvable knowledge (or less optimistically improvable ignorance):

* an unbound logic variable stands for no knowledge
* a variable may be unified with a value - representing certain knowledge that "something has some value"
* a variable may be unified with another variable, or a expression which is not fully bound - representing incomplete knowledge.

To start we will use a simple `String ` representation of variable ids:

Variables and bindings
----------------------

{% highlight scala %}
case class VarId (val id: String)
{% endhighlight %}

And we will use a standard scala `Map` for knowledge representation (variable bindings or substitutions):

{% highlight scala %}
case class Knowledge[T](map: Map[VarId, T])

object Knowledge {
  def empty[T] = Knowledge(Map.empty[VarId, T])
}
{% endhighlight %}

Unifiable
---------

We will define unification in terms of a type class like the authors of Polytpic Unification:

{% highlight scala %}
abstract class Unifiable[T] {
  /*
   * Subterms of T i.e 
   *
   * prolog equivalent      |   should return 
   * -----------------------+---------------------
   * some_functor(a, b, c)  |   [a, b, c]
   * 5,                     |   []
   * [a, b, c]              |   [a, [b, c]]
   *
   * IOW children should return the "shape" of the type or type constructor.
   */
  def children(t: T): List[T]

  /*
   * Some iff t can be projected into a VarId
   */
  def varCheck(t: T): Option[VarId]

  /*
   * Equality of the terms not including children
   *
   * LHS prolog equivalent  | RHS prolog equivalent  |  should return 
   * -----------------------|----+-------------------+-----------------
   * some_functor(a, b, c)  | some_functor(a, b, c)  | true
   * some_functor(a, b, c)  | some_functor(x, y, z)  | true
   * some_functor(a, b, c)  | some_functor(a, b)     | false
   * some_functor(a, b, c)  | other_functor(a, b, c) | false
   * 5                      | 5                      | true
   * 5                      | 4                      | false
   * [a, b, c]              | [a, b, c]              | true
   * [a, b, c]              | [a]                    | true 
   * [a, b, c]              | [a, b]                 | true
   * [a, b, c]              | [b, b]                 | true
   * [a, b, c]              | []                     | false
   * 5                      | functor(a, b, c)       | false
   *
   * topLevelEquivalent(t1, t2) iff topLevelEquivalent(t2, t1)
   */
  def topLevelEquivalent(t1: T, t2: T): Boolean

  /*
   * Enables lookups.
   */
  def mapChildren(t: T)(f: T => T): T
}
{% endhighlight %}

Understanding why `children` and `topLevelEquivalent` should behave as above
it's useful to look first at an example unifiable type, and the unification function itself.

A small `Universe`
------------------

{% highlight scala %}
sealed abstract class Universe
object Universe {
  case class UVar(v: VarId) extends Universe {
    def apply() = v
  }
  case class UInt(int: Int) extends Universe {
    def apply() = int
  }
  case object UNil extends Universe
  case class UCons(h: Universe, tail: Universe) extends Universe

  implicit val unifiable: Unifiable[Universe] = new Unifiable[Universe] {
    def children(t: Universe): List[Universe] = t match {
      case UInt(_) | UVar(_) | UNil=> 
        Nil
        
      case UCons(h, t) => List(h, t)
    }

    def mapChildren(t: Universe)(f: Universe => Universe): Universe = t match {
      case atomic @ (UInt(_) | UVar(_) | UNil) => 
        atomic

      case UCons(l, r) =>
        UCons(f(l), f(r))
    }

    def varCheck(t: Universe): Option[VarId] = t match {
      case UVar(v) => Some(v)
      case other => None
    }

    def topLevelEquivalent(l: Universe, r: Universe): Boolean = (l, r) match {
      case (UVar(l), UVar(r)) => l == r
      case (UInt(l), UInt(r)) => l == r
      case (UCons(_, _), UCons(_, _)) => true
      case (UNil, UNil) => true
      case _ => false
    }
  }
}
{% endhighlight %}

To implement unification we need to add some methods to `Knowledge`:

{% highlight scala %}
case class Knowledge[T](map: Map[VarId, T])(implicit uni: Unifiable[T]) {
  /*
   * Modify or introduce a binding `v <- t`
   */
  def modBind(v: VarId, t: T) = {
    copy(map.updated(v, t))
  }

  /*
   * Recursively looks up the value of variable `v` (if any)
   */
  def lookup(v: VarId): Option[T] = {
    map.lift(v).flatMap { t =>
      walk(t)
    }
  }

  /*
   * If `t` is a var or contains a var look it up recursively.
   */
  private[this] def walk(t: T): Option[T] = {
    uni.varCheck(t) match {
      case None =>
        Some(
          uni.mapChildren(t) { x =>
            // best effort
            walk(x).getOrElse(x)
          }
        )

      case Some(otherVar) =>
        lookup(otherVar)
    }
  }
}
{% endhighlight %}

Unification
-----------

Now we can, finally, implement unification:

{% highlight scala %}
object Unification {
  /*
   * Unify t1 with t2 assuming no knowledge.
   */
  def unify[T](t1: T, t2: T)(implicit unifiable: Unifiable[T]): Option[Knowledge[T]] = {
    unifyWith(t1, t2, Knowledge.empty[T])
  }

  /*
   * Unify t1 with t2 using/improving the passed knowledge.
   */
  def unifyWith[T](
      t1: T, 
      t2: T, 
      knowledge: Knowledge[T])(
      implicit unifiable: Unifiable[T]): Option[Knowledge[T]] = {
    def unifyTerms(l: T, r: T, k0: Knowledge[T]): Option[Knowledge[T]] = {
      val children = unifiable.children(l).zip(unifiable.children(r))

      children.foldLeft(Some(s0) : Option[Knowledge[T]]) { case (maybeK, (l, r)) =>
        maybeK.flatMap { knowledge =>
          unifyWith(l, r, knowledge)
        }
      }
    }

    (unifiable.varCheck(t1), unifiable.varCheck(t2)) match {
      case (None, None) => 
        if (unifiable.topLevelEquivalent(t1, t2)) {
          unifyTerms(t1, t2, knowledge)
        } else {
          None
        }

      case (Some(l), Some(r)) =>
        if (l == r) {
          Some(knowledge)
        } else {
          bind(l, t2, knowledge)
        }

      case (Some(l), None) =>
        bind(l, t2, knowledge)

      case (None, Some(r)) =>
        bind(r, t1, knowledge)
    }
  }

  def bind[T](
      v: VarId, 
      t: T, 
      knowledge: Knowledge[T])(
      implicit unifiable: Unifiable[T]): Option[Knowledge[T]] = {

    /*
     * Trying to unify `V` with `foo(V, b, c)` should fail as this is 
     * equivalent to infinite regress.
     */
    if (occursCheck(v, knowledge, t)) {
      None
    } else {
      knowledge.lookup(v) match {
        case None =>
          Some(knowledge.modBind(v, t))

        case Some(otherT) =>
          unifyWith(t, otherT, knowledge).map { s1 => s1.modBind(v, t) }
      }
    }
  }

  /*
   * Recursively checks if `v` occurs in `t` using the supplied knowledge.
   */
  def occursCheck[T](
      v: VarId, 
      knowledge: Knowledge[T], 
      t: T)(implicit unifiable: Unifiable[T]): Boolean = {
    def reachlist(l: List[VarId]): List[VarId] = l ::: l.flatMap(reachable)
    def reachable(v: VarId): List[VarId] = reachlist(
      knowledge.lookup(v).map { x => vars(x) }.getOrElse(List.empty)
    )

    // v may be aliased to some other variable 
    val aliased = knowledge.lookup(v).flatMap(unifiable.varCheck) match {
      case Some(alias) =>
        alias
      case None =>
        v
    }

    reachlist(vars(t)).contains(aliased)
  }

  private[this] def subTerms[T](t: T)(implicit unifiable: Unifiable[T]): List[T] = {
    t :: unifiable.children(t).flatMap { x => subTerms(x) }
  }

  private[this] def vars[T](t: T)(implicit unifiable: Unifiable[T]): List[VarId] = {
    subTerms(t).map(unifiable.varCheck).collect { case Some(v) => v }
  }
}
{% endhighlight %}

And a small test:

{% highlight scala %}
scala> def ulist(us: Universe*): Universe = us.foldRight(UNil : Universe)(UCons.apply)
def ulist(us: logprog.test.Universe*): logprog.test.Universe

scala>   val x: Universe = UVar(VarId("x"))
val x: logprog.test.Universe = UVar(VarId(x))

scala>   val y: Universe = UVar(VarId("y"))
val y: logprog.test.Universe = UVar(VarId(y))

scala>     import Unification._
import Unification._

scala>     import Universe._
import Universe._

scala>     val k1 = unifyWith(x, y, Knowledge.empty).get
val k1: logprog.Knowledge[logprog.test.Universe] = Knowledge(x -> UVar(VarId(y)))

scala>     val k2 = unifyWith(x, UInt(1), k1).get
val k2: logprog.Knowledge[logprog.test.Universe] = Knowledge(x -> UInt(1), y -> UInt(1))

scala>     val k3 = unifyWith(ulist(x, y), ulist(y, UInt(1)), Knowledge.empty).get
val k3: logprog.Knowledge[logprog.test.Universe] = Knowledge(x -> UVar(VarId(y)), y -> U Int(1))

scala>     println(k3.lookup(VarId("x")))
Some(UInt(1))

scala>     println(k3.lookup(VarId("y")))
Some(UInt(1))

scala>     val k0 = Knowledge.empty[Universe].
     |       modBind(VarId("v"), uvar("w")).modBind(VarId("w"), uvar("z"))
val k0: logprog.Knowledge[logprog.test.Universe] = Knowledge(v -> UVar(VarId(w)), w -> UVar(VarId(z)))

scala>     println("v in [1 | z]? " + Unification.occursCheck(VarId("v"), k0, UCons(UInt(1), uvar("z"))))
v in [1 | z]? true

scala>     println("z in [1 | v]? " + Unification.occursCheck(VarId("z"), k0, UCons(UInt(1), uvar("v"))))
z in [1 | v]? true

scala>     println("v in [1 | 2]? " + Unification.occursCheck(VarId("v"), k0, UCons(UInt(1), UInt(2))))
v in [1 | 2]? false
{% endhighlight %}

Logic Programming
=================

Our logic programming system will be implemented through (endo-)relations of `Knowledge` to `Knowledge`. We will call such relations goals or predicates, and start the implementation of our system with the primitive goal `unify`:

{% highlight scala %}
def unify[T : Unifiable](t1 : T, t2: T) = succeed[Knowledge[T]].flatMap { knowledge =>
  Unification.unifyWith(t1, t2, knowledge) match {
    case Some(newKnowledge) =>
      succeed.map(_ => newKnowledge)
    case None =>
      fail
  }
}
{% endhighlight %}

We will program the next goal `choice` against the concrete `Universe` type:

{% highlight scala %}
implicit class UniverseHasUnify(u: Universe) {
  def ===(v: Universe) = Relation.unify(u, v)
}

/*
 * Succeeds when `x` is a member of `list`.
 */
def choice(x: Universe, list: Universe): Relation[Knowledge[Universe], Knowledge[Universe]] = Relation.succeed[Knowledge[Universe]].flatMap { k =>
  list match {
    case UCons(h, t) => 
      (x === h) \/
      (choice(x, t))
    case other =>
      fail
  }
}
{% endhighlight %}

At this step we'll introduce some syntax allowing us to "run: predicates:

{% highlight scala %}
object Relation { 
  ...
  implicit class PredicateSyntax[T : Unifiable](p: Relation[Knowledge[T], Knowledge[T]]) {
    def run = p.apply(Knowledge.empty[T])
    def run1 = run.head
    def runAll = run.toList
  }
{% endhighlight %}

Running a predicate produces all of the results of the goal (the knowledge
obtained when reasoning).

{% highlight scala %}
scala>     println(choice(UInt(2), ulist(List(1, 2, 3).map(uint))).runAll)
List(Knowledge())
/* yes indeed, 2 is a member of [1, 2, 3] */

scala>     println(choice(UInt(10), ulist(List(1, 2, 3).map(UInt))).runAll)
List()
/* empty list corresponds to failure, no 10 is not a member of [1, 2, 3] */

scala>     println(choice(x, ulist(List(1, 2, 3).map(UInt))).runAll)
List(Knowledge(x -> UInt(1)), Knowledge(x -> UInt(2)), Knowledge(x -> UInt(3)))
/* x is a member of [1, 2, 3] then x is either 1, 2, or 3 */
{% endhighlight %}

Another predicate we can implement:

{% highlight scala %}
/*
 * x is the common element of l1 and l2.
 */
def commonElement(x: Universe, l1: Universe, l2: Universe) = Relation.succeed[Knowledge[Universe]].flatMap { _ =>
  choice(x, l1) /\
  choice(x, l2)
}
{% endhighlight %}

Because using and declaring predicates will be so common we could use with some syntax sugar:
{% highlight scala %}
object Relation { 
  ...
  type Predicate[T] = Relation[Knowledge[T], Knowledge[T]]
{% endhighlight %}

And because we actually never want to modify `Knowledge` in non-primitive
predicates we will use this method for defining predicates:

{% highlight scala %}
def defpred[T : Unifiable](p: Predicate[T]) = succeed[Knowledge[T]].flatMap { _ =>
  p
}
{% endhighlight %}

Which enables us to define commonElement as follows:

{% highlight scala %}
def commonElement(x: Universe, l1: Universe, l2: Universe) = defpred[Universe](
  choice(x, l1) /\
  choice(x, l2)
)
...
scala>     println(
     |       commonElement(
     |         x, 
     |         ulist(List(1, 2, 3).map(UInt)), 
     |         ulist(List(3, 4, 5).map(UInt))).runAll)
List(Knowledge(x -> UInt(3)))
/* the common element of [1, 2, 3] and [3, 4, 5] is 3 */

scala>     println(
     |       commonElement(
     |         x, 
     |         ulist(List(1, 2, 3).map(uint)), 
     |         ulist(List(3, 4, 1, 7).map(uint))).runAll)
List(Knowledge(x -> UInt(1)), Knowledge(x -> UInt(3)))
/* the common elements of [1, 2, 3], [3, 4, 1, 7] are 1 or 3 */

scala>     println(
     |       commonElement(
     |         x, 
     |         ulist(List(11, 2, 3).map(uint)), 
     |         ulist(List(13, 4, 1, 7).map(uint))).runAll)
List()
/* [11, 2, 3] and [13, 4, 1, 7] have no common elements */
{% endhighlight %}

Limitations
===========

I could go on with the definitions of a `cons` and `append` predicates, but even at this stage it's
apparent that the current implementation has some pretty serious limitations.

Firstly, the logical variables we use aren't scoped in any way - introducing
the variable `x` does so "globally".

Secondly, while we are able to write predicates against any arbitrary universe of types, implementing the actual predicates loses the benefit of scala's type system as the implemented predicates are [dynamic/unityped](https://existentialtype.wordpress.com/2011/03/19/dynamic-languages-are-static-languages/).

I will address these shortcomings in the next blog post.
