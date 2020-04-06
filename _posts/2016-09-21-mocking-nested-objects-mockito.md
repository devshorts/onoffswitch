---
layout: post
title: Mocking nested objects with mockito
date: 2016-09-21 22:55:16.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- mockito
- scala
- testing
meta:
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560218483;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4844;}i:1;a:1:{s:2:"id";i:4961;}i:2;a:1:{s:2:"id";i:4862;}}}}
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'

permalink: "/2016/09/21/mocking-nested-objects-mockito/"
---
Yes, I know its a code smell. But I live in the real world, and sometimes you need to mock nested objects. This is a scenario like:

[code]  
when(a.b.c.d).thenReturn(e)  
[/code]

The usual pattern here is to create a mock for each object and return the previous mock:

[code]  
val a = mock[A]  
val b = mock[B]  
val c = mock[C]  
val d = mock[D]

when(a.b).thenReturn(b)  
when(b.c).thenReturn(c)  
when(c.d).thenReturn(d)  
[/code]

But again, in the real world the signatures are longer, the types are nastier, and its never quite so clean. I figured I'd sit down and solve this for myself once and for all and came up with:

[scala]  
import org.junit.runner.RunWith  
import org.mockito.Mockito  
import org.scalatest.junit.JUnitRunner  
import org.scalatest.{FlatSpec, Matchers}

@RunWith(classOf[JUnitRunner])  
class Tests extends FlatSpec with Matchers {  
 "Mockito" should "proxy nested objects" in {  
 val parent = Mocks.mock[Parent]

Mockito.when(  
 parent.  
 mock(\_.getChild1).  
 mock(\_.getChild2).  
 mock(\_.getChild3).  
 value.doWork()  
 ).thenReturn(3)

parent.value.getChild1.getChild2.getChild3.doWork() shouldEqual 3  
 }  
}

class Child3 {  
 def doWork(): Int = 0  
}

class Child2 {  
 def getChild3: Child3 = new Child3  
}

class Child1 {  
 def getChild2: Child2 = new Child2  
}

class Parent {  
 def getChild1: Child1 = new Child1  
}  
[/scala]

As you can see in the full test we can create some mocks object, and reference the call chain via extractor methods.

The actual mocker is really pretty simple, it just looks nasty cause of all the lambdas/manifests. All thats going on here is a way to pass the next object to a chain and extract it with a method. Then we can create a mock using the manifest and assign that mock to the source object via the lambda.

[scala]  
import org.mockito.Mockito

object Mocks {  
 implicit def mock[T](implicit manifest: Manifest[T]) = new RichMockRoot[T]

class RichMockRoot[T](implicit manifest: Manifest[T]) {  
 val value = Mockito.mock[T](manifest.runtimeClass.asInstanceOf[Class[T]])

def mock[Y](extractor: T =\> Y)(implicit manifest: Manifest[Y]): RichMock[Y] = {  
 new RichMock[T](value, List(value)).mock(extractor)  
 }  
 }

class RichMock[T](c: T, prevMocks: List[\_]) {  
 def mock[Y](extractor: T =\> Y)(implicit manifest: Manifest[Y]): RichMock[Y] = {  
 val m = Mockito.mock[Y](manifest.runtimeClass.asInstanceOf[Class[Y]])

Mockito.when(extractor(c)).thenReturn(m)

new RichMock(m, prevMocks ++ List(m))  
 }

def value: T = c

def mockChain[Y](idx: Int) = prevMocks(idx).asInstanceOf[Y]

def head[Y] = mockChain[Y](0)  
 }  
}  
[/scala]

The main idea here is just to hide away the whole "make b and have it return c" for you. You can even capture all the intermediate mocks in a list (I called it a mock chain), and expose the first element of the list with `head`. With a little bit of scala manifest magic you can even get around needing to pass class files around and can leverage the generic parameter (boy, feels almost like .NET!).

