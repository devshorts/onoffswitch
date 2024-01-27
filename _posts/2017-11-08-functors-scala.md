---
layout: post
title: Functors in scala
date: 2017-11-08 01:59:09.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- functional
- scala
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _syntaxhighlighter_encoded: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560455198;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:7777;}i:1;a:1:{s:2:"id";i:4919;}i:2;a:1:{s:2:"id";i:3565;}}}}
  _wpcom_is_markdown: '1'
  _jetpack_dont_email_post_to_subs: '1'

permalink: "/2017/11/08/functors-scala/"
---
A coworker of mine and I frequently talk about higher kinded types, category theory, and lament about the lack of unified types in scala: namely functors. A functor is a fancy name for a thing that can be mapped on. Wanting to abstract over something that is mappable comes up more often than you think. I don't necessarily care that its an Option, or a List, or a whatever. I just care that it has a map.

We're not the only ones who want this. Cats, Shapeless, Scalaz, all have implementations of functor. The downside there is that usually these definitions tend to leak throughout your ecosystem. I've written before about ecosystem and library management, and it's an important thing to think about when working at a company of 50+ people. You need to think long and hard about putting dependencies on things. Sometimes you can, if those libraries have good versioning or back-compat stories, or if they expose lightweight API's with heavyweight bindings that you can separate out.

Often times these libraries aren't really well suited for large scale use and so you're forced to either replicate, denormalize, or otherwise hide away how those things come into play.

In either case, this post isn't about that. I just wanted to know how the hell those libraries did the magic.

Let me lay out the final product first and we'll break it down:

```scala
  
trait Functor[F[\_]] {  
 def map[A, B](f: F[A])(m: A =\> B): F[B]  
}

object Functor {  
 implicit class FunctorOps[F[\_], A](f: F[A])(implicit functor: Functor[F]) {  
 def map[B](m: A =\> B): F[B] = {  
 functor.map(f)(m)  
 }  
 }

implicit def iterableFunctor[T[X] \<: Traversable[X]] = new Functor[T] {  
 override def map[A, B](f: T[A])(m: A =\> B) = {  
 f.map(m).asInstanceOf[T[B]]  
 }  
 }

implicit def optionFunctor = new Functor[Option] {  
 override def map[A, B](f: Option[A])(m: A =\> B) = {  
 f.map(m)  
 }  
 }

implicit def futureFunctor(implicit executionContext: ExecutionContext) = new Functor[Future] {  
 override def map[A, B](f: Future[A])(m: A =\> B) = {  
 f.map(m)  
 }  
 }  
}  

```

And no code is complete without a test...

```scala
  
class Tests extends FlatSpec with Matchers {

import com.curalate.typelevel.Functor  
 import com.curalate.typelevel.Functor.\_

private def testMaps[T[\_] : Functor](functor: T[Int]): T[Int] = {  
 functor.map(x =\> x + 1)  
 }

"A test" should "run" in {  
 testMaps(List(1)) shouldEqual List(2)

testMaps(Some(1): Option[Int]) shouldEqual Some(2)

testMaps(None: Option[Int]) shouldEqual None

testMaps(Set(1)) shouldEqual Set(2)

Await.result(testMaps(Future.successful(1)), Duration.Inf) shouldEqual 2  
 }  
}  

```

How did we get here? First if you look at the definition of functor again

```scala
  
trait Functor[F[\_]] {  
 def map[A, B](f: F[A])(m: A =\> B): F[B]  
}  

```

We're saying that

1. Given a type F that contains some other unknown type (i.e. F is a box, like List, or Set)
2. Define a map function from A to B and give me back a type of F of B

The nuanced part here is that the map takes an instance of `F[A]`. We need this to get all the types to be happy, since we have to specify somewhere that `F[A]` and `A => B` are paired together.

Lets make a functor for list, since that one is pretty easy:

```scala
  
object Functor {  
 implicit lazy val listFunctor = new Functor[List] {  
 override def map[A, B](f: List[A])(m: A =\> B) = {  
 f.map(m)  
 }  
 }  
}  

```

Now we can get an instance of functor from a `List[T]`

We could use it like this now:

```scala
  
def listMapper(f: Functor[List[Int]])(l: List[Int]) = {  
 f.map(l)(\_ + 1)  
}  

```

But that sort of sucks. I don't want to know I have a list, that defeats the purpose of a functor!

What if we do

```scala
  
def intMapper[T[\_]](f: Functor[T[Int]])(l: T[Int]) = {  
 f.map(l)(\_ + 1)  
}  

```

Kind of better. Now I have a higher kinded type that doesn't care about what the box is. But I still need to somehow _get_ an instance of a functor to do my mapping.

This is where the `ops` class come in:

```scala
  
implicit class FunctorOps[F[\_], A](f: F[A])(implicit functor: Functor[F]) {  
 def map[B](m: A =\> B): F[B] = {  
 functor.map(f)(m)  
 }  
}  

```

This guy says _given a container, and a functor for that container, here is a helpful map function_. It's giving us an extension method on `F[A]` that adds `map`. You may wonder, well dont' all things we're mapping on already have a map function? And the answer is yes, but the compiler doesn't _know_ that since we're dealing with only generics here!

Now, we can import our functor ops class and finally get that last bit to work:

```scala
  
class Tests extends FlatSpec with Matchers {

import com.curalate.typelevel.Functor  
 import com.curalate.typelevel.Functor.\_

private def testMaps[T[\_] : Functor](functor: T[Int]): T[Int] = {  
 functor.map(x =\> x + 1)  
 }

"A test" should "run" in {  
 testMaps(List(1)) shouldEqual List(2)

testMaps(Some(1): Option[Int]) shouldEqual Some(2)

testMaps(None: Option[Int]) shouldEqual None

testMaps(Set(1)) shouldEqual Set(2)

Await.result(testMaps(Future.successful(1)), Duration.Inf) shouldEqual 2  
 }  
}  

```

Pulling it all together, we're asking for a type of `T` that is a box of anything that has an implicit `Functor[T]` typeclass. We want to use the `map` method on the functor of `T` and that map method comes because we leverage the implicit `FunctionOps`.

It helps to think of `functor` not as an interface that a thing implements, but as a typeclass/extension of a thing. I.e. in order to get a map, you have to wrap something.

Anyways, big thanks to Christian for helping me out.

