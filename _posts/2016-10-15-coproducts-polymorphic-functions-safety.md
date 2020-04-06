---
layout: post
title: Coproducts and polymorphic functions for safety
date: 2016-10-15 22:58:45.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- coproduct
- scala
- shapeless
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpcom_is_markdown: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1555115004;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4862;}i:1;a:1:{s:2:"id";i:4961;}i:2;a:1:{s:2:"id";i:4905;}}}}
  _wpas_done_all: '1'
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/10/15/coproducts-polymorphic-functions-safety/"
---
I was recently exploring [shapeless](https://github.com/milessabin/shapeless) and a coworker turned me onto the interesting features of coproducts and how they can be used with polymorphic functions.

Frequently when using pattern matching you want to make sure that all cases are exhaustively checked. A non exhaustive pattern match is a runtime exception waiting to happen. As a scala user, I'm all about compile time checking. For classes that I own I can enforce exhaustiveness by creating a sealed trait heirarchy:

[code lang=scala]  
sealed trait Base  
case class Sub1() extends Base  
case class Sub2() extends Base  
[/code]

And if I ever try and match on an `Base` type I'll get a compiler warning (that I can fail on) if all the types aren't matched. This is nice because if I ever add another type, I'll get a (hopefully) failed build.

But what about the scenario where you _don't_ own the types?

[code lang=scala]  
case class Type1()  
case class Type2()  
case class Type3()  
[/code]

They're all completely unrelated. Even worse is how do you create a generic function that accepts an instance of those 3 types but no others? You could always create overloaded methods:

[code lang=scala]  
def takesType(type: Type1) = ???  
def takesType(type: Type2) = ???  
def takesType1(type: Type3) = ???  
[/code]

Which works just fine, but what if that type needs to be passed through a few layers of function calls before its actually acted on?

[code lang=scala]  
def doStuff(type: Type1) = ... takesType(type1)  
def doStuff(type: Type2) = ... takesType(type2)  
def doStuff(type: Type3) = ... takesType(type3)  
[/code]

Oh boy, this is a mess. We can't get around with just using generics with type bounds since there is no unified type for these 3 types. And even worse is if we add another type. We could use an either like `Either[Type1, Either[Type2, Either[Type3, Nothing]]]`

Which lets us write just one function and then we have to match on the subsets. This is kind of gross too since its polluted with a bunch of eithers. Turns out though, that a coproduct is exactly this... a souped up either!

Defining

[code lang=scala]  
type Items = Type1 :+: Type2 :+: Type3 :+: CNil  
[/code]

(where CNil is the terminator for a coproduct) we now have a unified type for our collection. We can write functions like :

[code lang=scala]  
def doStuff(item: Items) = {  
 // whatever  
 takesType(item)  
}  
[/code]

At some point, you need to lift an instance of `Type1` etc into a type of `Item` and this can be done by calling `Coproduct[Item](instance)`. This call will fail to compile if the type of the instance is not a type of `Item`. You also are probably going to want to actually do work with the thing, so you need to unbox this souped up either and do stuff with it

This is where the shapeless `PolyN` methods come into play.

[code lang=scala]  
object Worker {  
 type Items = Type1 :+: Type2 :+: Type3 :+: CNil

object thisIsAMethod extends Poly1 {  
 // corresponding def for the data type of the coproduct instance  
 implicit def invokedOnType1 = at[Type1](data =\> data.toString)  
 implicit def invokedOnType2 = at[Type2](data =\> data.toString)  
 implicit def invokedOnType3 = at[Type3](data =\> data.toString)  
 }

def takesItem(item: Item): String = {  
 thisIsAMethod(item)  
 }  
}

class Provider {  
 Worker.takesItem(Coproduct[Item](Type1()) // ok  
 Worker.takesItem(Coproduct[Item](WrongType()) // fails  
}

[/code]

The object `thisIsAMethod` creates a bunch of implicit type dependent functions that are defined at all the elements in the coproduct. If we add another option to our coproduct list, we'll get a compiler error when we try and use the coproduct against the polymorphic function. This accomplishes the same thing as giving us the exhaustiveness check but its an even stronger guarantee as the build will fail.

While it is a lot of hoops to jump through, and can be a little mind bending, I've found that coproducts and polymorphic functions are a really nice addition to my scala toolbox. Being able to strongly enforce these kinds of contracts is pretty neat!

