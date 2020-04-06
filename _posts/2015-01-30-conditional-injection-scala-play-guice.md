---
layout: post
title: Conditional injection with scala play and guice
date: 2015-01-30 01:08:21.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dependency injection
- guice
- java
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1557884187;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4939;}i:1;a:1:{s:2:"id";i:4961;}i:2;a:1:{s:2:"id";i:4919;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/01/30/conditional-injection-scala-play-guice/"
---
It's been a crazy year for me. For those who don't know I moved from the east coast to the west coast to work for a rather _large_ softare company in seattle (I'll let you figure which one out) and after a few short weeks realized I made a horrible mistake and left the team. I then found a cool job at a smaller .net startup that was based in SF and met some awesome people and learned a lot. But, I've been poached by an old coworker and am now going to go work at a place that uses more open source things so I decided to kick into gear and investigate scala and play.

For the most part I'm doing a mental mapping of .NET's web api framework to the scala play framework, but the more I play in play (pun intended) the more I like it.

On one of my past projects a coworker of mine set up a really interesting framework leveraging ninject and web api where you can conditionally inject a data source for test data by supplying a query parameter to a rest API of "test". So the end result looks something like:

[csharp]  
[GET("foo/{name}")]  
public void GetExample(string name, [IDataSource] dataProvider){  
 // act on data provider  
}  
[/csharp]

The way it chose the correct data provider is by leveraging a custom parameter binder that will resolve the source from the ninject kernel based on the query parameters. I've found that this worked out really well in practice. It lets the team set up some sample data while testers/qa/ui devs can start building out consuming code before the db layers are even complete.

I really liked working with this pattern so I wanted to see how we can map this to the scala play framework. Forgive me if what I post isn't idiomatic scala, I've only been at it for a day :)

First I want to define some data sources

[scala]  
trait DataSource{  
 def get : String  
}

class ProdSource extends DataSource{  
 override def get: String = "prod"  
}

class TestSource extends DataSource {  
 override def get : String = "test"  
}  
[/scala]

It should be pretty clear whats going on here. I've defined two classes that implement the data source trait. Which one that gets injected should be defined by a query parameter.

Guice lets you define bindings for the same trait (interface) to a target class based on "keys". What this means is you can say "_give me class A, and use the default binding_", or you can say "_give me class A, but the one that is tagged with interface Test_". When you register the classes you can provider this extra tagging mechanism. This is going to be useful because you can now request different versions of the interface from the binding kernel.

Lets just walk through the remaining example. First we need the interface, but Guice wants it to be an annotation. Since scala has weird support for annotations and the JVM has shitty type erasure, I had to write the annotation in java

[java]  
import com.google.inject.BindingAnnotation;

import java.lang.annotation.ElementType;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;

@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@BindingAnnotation  
public @interface TestAnnotation {}  
[/java]

I'm honestly not even sure I need the @Target, but whatever.

Next we're gonna create some binding modules for Guice to use where we can specify the conditional binding:

[scala]  
package Modules

import annotations.TestAnnotation  
import com.google.inject.AbstractModule  
import controllers.{DataSource, ProdSource, TestSource}

class SourceModule extends AbstractModule {

override def configure(): Unit = {  
 testable(classOf[DataSource], classOf[ProdSource], classOf[TestSource])  
 }

def testable[TInterface, TMain \<: TInterface, TDev \<: TInterface](  
 interface: Class[TInterface],  
 main: Class[TMain],  
 test: Class[TDev]) = {  
 val markerClass = classOf[TestAnnotation]

bind(interface).to(main)

bind(interface) annotatedWith markerClass to test  
 }  
}  
[/scala]

What this is saying is that given the 3 types (the main interface, the implementation of the main item, and the implementation of the dev item) to conditionally bind the dev item to the marker class of "TestAnnotation". This will make sense when you see how its used.

As normal, guice is used to set up the controller instantation with the source module registered.

[scala]  
import Modules.{DbModule, SourceModule}  
import com.google.inject.Guice  
import play.api.GlobalSettings

object Global extends GlobalSettings {

val kernel = Guice.createInjector(new SourceModule())

override def getControllerInstance[A](controllerClass: Class[A]): A = {  
 kernel.getInstance(controllerClass)  
 }  
}  
[/scala]

Now comes the fun part of actually resolving the query parameter. I'm going to wrap an action and create a new action so we can get a nodejs style `(datasource, request) =>` lambda.

[scala]  
trait Sourceable{  
 val kernelSource : Injector

def WithSource[T] (clazz : Class[T]) (f: ((T, Request[AnyContent]) =\> Result)) : Action[AnyContent] = {  
 Action { request =\> {  
 val binder =  
 request.getQueryString(sourceableQueryParamToggle) match {  
 case Some(\_) =\> kernelSource.getInstance(Key.get(clazz, classOf[TestAnnotation]))  
 case None =\> kernelSource.getInstance(clazz)  
 }

f(binder, request)  
 }}  
 }

def sourceableQueryParamToggle = "test"  
}  
[/scala]

The kernel never has to be registered since Guice will auto inject it when its asked for (its implicity available). Whats happening here is that we set up the kernel and the target interface type we want to get (i.e. DataSource). If the query string matches the sourceable query param toggle (i.e. the word "test") then it'll pick up the registered data source using the "test annotation" marker. Otherwise it uses the default.

Finally the controller now looks like this:

[scala]  
@Singleton  
class Application @Inject() (db : DbAccess, kernel : Injector) extends Controller with Sourceable {  
 override val kernelSource: Injector = kernel

def binding(name : String) = WithSource(classOf[DataSource]){ (provider, request) =\>  
 {  
 val result = name + ": " + provider.get

Ok(result)  
 }}  
}  
[/scala]

And the route

[scala]  
GET /foo/:name @controllers.Application.binding(name: String)  
[/scala]

The kernel value is provided to the trait and any other methods can now ask for a data provider of a particular type and get it.

Full source available at my [github](https://github.com/devshorts/scala-injector).

