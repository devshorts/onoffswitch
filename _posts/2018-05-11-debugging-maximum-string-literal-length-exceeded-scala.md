---
layout: post
title: Debugging "Maximum String literal length exceeded" with scala
date: 2018-05-11 01:02:19.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- bug
- scala
meta:
  _edit_last: '1'
  _wpcom_is_markdown: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560002911;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:4862;}i:2;a:1:{s:2:"id";i:4068;}}}}
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'

permalink: "/2018/05/11/debugging-maximum-string-literal-length-exceeded-scala/"
---
Today I ran into a fascinating bug. We use [ficus](https://github.com/iheartradio/ficus) as a HOCON auto parser for scala. It works great, because parsing configurations into strongly typed case classes is annoying. Ficus works by using a macro to invoke implicitly in scope `Reader[T]` classes for data types and recursively builds the nested parser.

I went to create a test for a new custom field I added to our config:

[scala]  
class ProductConfigTests extends FlatSpec {  
 "Configs" should "be valid in QA" in {  
 assert(ConfigLoader.verify(ProductsConfig, Environment.QA).isSuccess)  
 }  
}  
[/scala]

Our config verifier just invokes the hocon parser and makes sure it doesn't throw an error. `ProductsConfig` has a lot of fields to it, and I recently added a new one. Suddenly the test broke with the following error:

[code]  
error] Error while emitting com/services/products/service/tests/ConfigTests  
[error] Maximum String literal length exceeded  
[error] one error found  
[error] (server/test:compileIncremental) Compilation failed  
[error] Total time: 359 s, completed May 10, 2018 4:56:02 PM  
\> test:compile  
[info] Compiling 36 Scala sources to /Users/antonkropp/src/products/server/target/scala-2.12/test-classes...  
java.lang.IllegalArgumentException: Maximum String literal length exceeded  
 at scala.tools.asm.ByteVector.putUTF8(ByteVector.java:213)  
 at scala.tools.asm.ClassWriter.newUTF8(ClassWriter.java:1114)  
 at scala.tools.asm.ClassWriter.newString(ClassWriter.java:1582)  
 at scala.tools.asm.ClassWriter.newConstItem(ClassWriter.java:1064)  
 at scala.tools.asm.MethodWriter.visitLdcInsn(MethodWriter.java:1187)  
 at scala.tools.asm.tree.LdcInsnNode.accept(LdcInsnNode.java:71)  
 at scala.tools.asm.tree.InsnList.accept(InsnList.java:162)  
 at scala.tools.asm.tree.MethodNode.accept(MethodNode.java:820)  
 at scala.tools.asm.tree.MethodNode.accept(MethodNode.java:730)  
[/code]

Wat?

I fired up `sbt -jvm-debug 5005` and attached to the compiler.

![](http://onoffswitch.net/wp-content/uploads/2018/05/maxstring1.png)

I can def see that there is some sort of class being written with a large const string. But why? I'd never seen this before.

I went to another service that has a test exactly like this for its config and used [cfr](http://www.benf.org/other/cfr/) to decompile the generated scala files:

[code]  
antonkropp at combaticus in ~/src/curalate/queue-batcher/server/target/scala-2.12/test-classes/com/curalate/services/queuebatcher/service/tests (devx/minimize-io-calls)  
$ ls  
total 1152  
-rw-r--r-- 1 antonkropp staff 47987 May 10 11:20 BatchTrackerTests.class  
-rw-r--r-- 1 antonkropp staff 145293 May 10 13:34 BitGroupTests.class  
-rw-r--r-- 1 antonkropp staff 3112 May 9 13:23 ConfigTests$$anon$1$$anon$2$$anon$3.class  
-rw-r--r-- 1 antonkropp staff 3553 May 9 13:23 ConfigTests$$anon$1$$anon$2$$anon$4$$anon$5.class  
-rw-r--r-- 1 antonkropp staff 5190 May 9 13:23 ConfigTests$$anon$1$$anon$2$$anon$4.class  
-rw-r--r-- 1 antonkropp staff 3627 May 9 13:23 ConfigTests$$anon$1$$anon$2$$anon$6.class  
-rw-r--r-- 1 antonkropp staff 3906 May 9 13:23 ConfigTests$$anon$1$$anon$2.class  
-rw-r--r-- 1 antonkropp staff 4904 May 9 13:23 ConfigTests$$anon$1$$anon$7.class  
-rw-r--r-- 1 antonkropp staff 4598 May 9 13:23 ConfigTests$$anon$1$$anon$8.class  
-rw-r--r-- 1 antonkropp staff 5063 May 9 13:23 ConfigTests$$anon$1.class  
-rw-r--r-- 1 antonkropp staff 3125 May 9 13:23 ConfigTests$$anon$17$$anon$18$$anon$19.class  
-rw-r--r-- 1 antonkropp staff 3573 May 9 13:23 ConfigTests$$anon$17$$anon$18$$anon$20$$anon$21.class  
-rw-r--r-- 1 antonkropp staff 5213 May 9 13:23 ConfigTests$$anon$17$$anon$18$$anon$20.class  
-rw-r--r-- 1 antonkropp staff 3640 May 9 13:23 ConfigTests$$anon$17$$anon$18$$anon$22.class  
-rw-r--r-- 1 antonkropp staff 3924 May 9 13:23 ConfigTests$$anon$17$$anon$18.class  
-rw-r--r-- 1 antonkropp staff 4914 May 9 13:23 ConfigTests$$anon$17$$anon$23.class  
-rw-r--r-- 1 antonkropp staff 4606 May 9 13:23 ConfigTests$$anon$17$$anon$24.class  
-rw-r--r-- 1 antonkropp staff 5073 May 9 13:23 ConfigTests$$anon$17.class  
-rw-r--r-- 1 antonkropp staff 3119 May 9 13:23 ConfigTests$$anon$9$$anon$10$$anon$11.class  
-rw-r--r-- 1 antonkropp staff 3566 May 9 13:23 ConfigTests$$anon$9$$anon$10$$anon$12$$anon$13.class  
-rw-r--r-- 1 antonkropp staff 5205 May 9 13:23 ConfigTests$$anon$9$$anon$10$$anon$12.class  
-rw-r--r-- 1 antonkropp staff 3634 May 9 13:23 ConfigTests$$anon$9$$anon$10$$anon$14.class  
-rw-r--r-- 1 antonkropp staff 3915 May 9 13:23 ConfigTests$$anon$9$$anon$10.class  
-rw-r--r-- 1 antonkropp staff 4909 May 9 13:23 ConfigTests$$anon$9$$anon$15.class  
-rw-r--r-- 1 antonkropp staff 4601 May 9 13:23 ConfigTests$$anon$9$$anon$16.class  
-rw-r--r-- 1 antonkropp staff 5066 May 9 13:23 ConfigTests$$anon$9.class  
-rw-r--r-- 1 antonkropp staff 87180 May 9 13:23 ConfigTests.class  
-rw-r--r-- 1 antonkropp staff 69451 May 9 13:23 DbTests.class  
-rw-r--r-- 1 antonkropp staff 12985 May 9 13:23 MysqlTests.class  
-rw-r--r-- 1 antonkropp staff 68418 May 10 12:40 Tests.class  
drwxr-xr-x 4 antonkropp staff 128 May 9 13:23 db  
drwxr-xr-x 9 antonkropp staff 288 May 9 13:23 modules

$ java -jar ~/tools/cfr.jar ConfigTests.class  
[/code]

1000 lines later, I can see

![](http://onoffswitch.net/wp-content/uploads/2018/05/maxstring2.png)

So something is putting in a large string of the configuration parser compiled into the class file.

I checked the ficus source code and its not it, so it must be something with the test.

Turns out `assert` is a macro from scalatest:

[scala]  
def assert(condition: Boolean)(implicit prettifier: Prettifier, pos: source.Position): Assertion = macro AssertionsMacro.assert  
[/scala]

Where the macro  
[scala]  
def assert(context: Context)(condition: context.Expr[Boolean])(prettifier: context.Expr[Prettifier], pos: context.Expr[source.Position]): context.Expr[Assertion] =  
 new BooleanMacro[context.type](context, "assertionsHelper").genMacro[Assertion](condition, "macroAssert", context.literal(""), prettifier, pos)

[/scala]  
Is looking for an implicit position.

Position is from scalactic which comes with scalatest

[scala]  
case class Position(fileName: String, filePathname: String, lineNumber: Int)

/\*\*  
 \* Companion object for \<code\>Position\</code\> that defines an implicit  
 \* method that uses a macro to grab the enclosing position.  
 \*/  
object Position {

import scala.language.experimental.macros

/\*\*  
 \* Implicit method, implemented with a macro, that returns the enclosing  
 \* source position where it is invoked.  
 \*  
 \* @return the enclosing source position  
 \*/  
 implicit def here: Position = macro PositionMacro.genPosition  
}  
[/scala]

And here we can ascertain that the macro expansion of the ficus config parser is being captured by the position file macro and auto compiled into the _assert_ statement!

Changing the test to be

[scala]  
class ProductConfigTests extends FlatSpec {  
 "Configs" should "be valid in QA" in {  
 validate(ConfigLoader.verify(ProductsConfig, Environment.QA).isSuccess)  
 }

/\*\*  
 \* This validate function needs to exist because this bug is amazing.  
 \*  
 \* `assert` is a macro from scalatest that automatically compiles the contextual source tree  
 \* into the assert, so that you can get line number and metadata context if the line fails.  
 \*  
 \* The ficus macro expander for ProductConfig is larger than 65k characters, which is normally fine  
 \* for code, however since scalatest tries to compile this \> 65k anonymous class tree as a \_string\_  
 \* it breaks the java compiler!  
 \*  
 \* By breaking the function scope and having the macro create a closure around the \_validate\_ block  
 \* it no longer violates the 65k static string constraint  
 \*/  
 private def validate(block: =\> Boolean): Unit = {  
 assert(block)  
 }  
}  
[/scala]

Now makes the test pass. What a day.

