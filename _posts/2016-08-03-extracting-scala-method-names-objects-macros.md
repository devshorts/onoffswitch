---
layout: post
title: Extracting scala method names from objects with macros
date: 2016-08-03 00:33:53.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- ast
- macro
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560513278;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4991;}i:1;a:1:{s:2:"id";i:4575;}i:2;a:1:{s:2:"id";i:4905;}}}}
  _wpas_done_all: '1'
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/08/03/extracting-scala-method-names-objects-macros/"
---
I have a soft spot in me for AST's ever since I went through the exercise of [building](http://onoffswitch.net/building-a-custom-lexer/) my own [language](https://github.com/devshorts/LanguageCreator). Working in Java I missed the dynamic ability to get compile time information, though I knew it was available as part of the annotation processing pipleine during compilation (which is how lombok works). Scala has something similiar in the concept of macros: a way to hook into the compiler, manipulate or inspect the syntax tree, and rewrite or inject whatever you want. It's a wonderfully elegant system that reminds me of Lisp/Clojure macros.

I ran into a situation (as always) where I really wanted to get the name of a function dynamically. i.e.

[scala]  
class Foo {  
 val field: String = ""  
 def method(): Unit = {}  
}

val name: String = ??.field // = "field"  
[/scala]

In .NET this is pretty easy since at runtime you can create an expression tree which gives you the AST. But I haven't been in .NET in a while, so off to macros I went!

First off, I found the documentation regarding macros to be lackluster. It's either rudimentary with trivial examples, or the learning curve was steep and I was too lazy to read through all of it. Usually when I encounter scenarios like this I turn to exploratory programming, where I have a unit test that sets up a basic example and I leverage the debugger and intellij live REPL to poke through what I can and can't do. Time to get set up.

First, I needed to create a new submodule in my multi module maven project that would contain my macro. The reason is that you can't use macros in the same compilation unit that they are defined in. You can however, use macros in a macros test since the compiler compiles test sources different from regular sources.

That said, debugging macros is harder than normal because you aren't debugging your running program, you are debugging the actual compiler. I found this [blog post](http://www.cakesolutions.net/teamblogs/2013/09/30/debugging-scala-macros) which was a life saver, even though it was missing a few minor pieces.

1. Set the main class to `scala.tools.nsc.Main`  
2. Set the VM args to `-Dscala.usejavacp=true`  
3. Set the program arguments to first point to the file containing the macro, then the file to compile that uses the macro:

> -cp types.Types macros/src/main/scala/com/devshorts/common/macros/MethodNames.scala config/src/test/scala/config/ConfigProxySpec.scala

Now you can actually debug your macro!

First let me show the test

[scala]  
case class MethodNameTest(field1: Object) {  
 def getFoo(arg: Object): Unit = {}  
 def getFoo2(arg: Object, arg2: Object): Unit = {}  
}

class MethodNamesMacroSpec extends FlatSpec with Matchers {  
 "Names macro" should "extract from an function" in {  
 methodName[MethodNameTest](\_.field1) shouldEqual MethodName("field1")  
 }

it should "extract when the function contains an argument" in {  
 methodName[MethodNameTest](\_.getFoo(null)) shouldEqual MethodName("getFoo")  
 }

it should "extract when the function contains multiple argument" in {  
 methodName[MethodNameTest](\_.getFoo2(null, null)) shouldEqual MethodName("getFoo2")  
 }

it should "extract when the method is curried" in {  
 methodName[MethodNameTest](m =\> m.getFoo2 \_) shouldEqual MethodName("getFoo2")  
 }  
}  
[/scala]

![macro](http://onoffswitch.net/wp-content/uploads/2016/08/macro.png)

`methodName` here is a macro that extracts the method name from a lambda passed in of the parameterized generic type. What's nice about how scala set up their macros is you provide an alias for your macro such that you can re-use the macro but type it however you want.

[scala]  
object MethodNames {  
 implicit def methodName[A](extractor: (A) =\> Any): MethodName = macro methodNamesMacro[A]

def methodNamesMacro[A: c.WeakTypeTag](c: Context)(extractor: c.Expr[(A) =\> Any]): c.Expr[MethodName] = {  
 ...  
 }  
}  
[/scala]

I've made the methodName function take a generic and a function that uses that generic (even though no actual instance is ever passed in). The nice thing about this is I can re-use the macro typed as another function elsewhere. Imagine I want to pin `[A]` so people don't have to type it. I can do exactly that!

[scala]  
case class Configuration (foo: String)

implicit def config(extractor: Configuration =\> Any): MethodName = macro MethodNames.methodNamesMacro[Configuration]

config(\_.foo) == "foo"  
[/scala]

At this point its time to build the bulk of the macro. The idea is to inspect parts of the AST and potentially walk it to find the pieces we want. Here's what I ended up with:

[scala]  
def methodNamesMacro[A: c.WeakTypeTag](c: Context)(extractor: c.Expr[(A) =\> Any]): c.Expr[MethodName] = {  
 import c.universe.\_

@tailrec  
 def resolveFunctionName(f: Function): String = {  
 f.body match {  
 // the function name  
 case t: Select =\> t.name.decoded

case t: Function =\>  
 resolveFunctionName(t)

// an application of a function and extracting the name  
 case t: Apply if t.fun.isInstanceOf[Select] =\>  
 t.fun.asInstanceOf[Select].name.decoded

// curried lambda  
 case t: Block if t.expr.isInstanceOf[Function] =\>  
 val func = t.expr.asInstanceOf[Function]

resolveFunctionName(func)

case \_ =\> {  
 throw new RuntimeException("Unable to resolve function name for expression: " + f.body)  
 }  
 }  
 }

val name = resolveFunctionName(extractor.tree.asInstanceOf[Function])

val literal = c.Expr[String](Literal(Constant(name)))

reify {  
 MethodName(literal.splice)  
 }  
}  
[/scala]

For more details on parts of the AST [here is a great resource](https://github.com/wolfe-pack/wolfe/wiki/Scala-AST-reference)

In the first case, when we pass in `methodName[Config](_.method)` it gets mangled into a function with a body that is of `x$1.method`. The select indicates the x$1 instance and selects the `method` expression of it. This is an easy case.

In the block case that maps to when we call `methodName[Config](c => c.thing _)`. In this case we have a function but its curried. In this scenario the function body is a block who's inner expression is a function. But, the functions body of that inner function is an Apply.

> Apply takes two arguments -- a Select or an Ident for the function and a list of arguments

So that makes sense.

The rest is just helper methods to recurse.

The last piece of the puzzle is to create an instance of a string literal and splice it into a new expression returning the `MethodName` case class that contains the string literal.

All in all a fun afternoons worth of code and now I get semantic string safety. A great use case here can be to type configuration values or other string semantics with a trait. You can get compile time refactoring + type safety. Other use cases are things like database configurations and drivers (the .NET mongo driver uses expression trees to type an object to its underlying mongo collection).

