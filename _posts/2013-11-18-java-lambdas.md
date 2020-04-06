---
layout: post
title: Java lambdas
date: 2013-11-18 08:00:12.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- java
- lambdas
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560197513;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4627;}i:1;a:1:{s:2:"id";i:4596;}i:2;a:1:{s:2:"id";i:4316;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/11/18/java-lambdas/"
---
I'm not a java person. I've never used it in production, nor have I spent any real time with it outside of my professional work. However, when a language dawns upon lambdas I am drawn to try out their implementation. I've long since despised Java for the reasons of verbosity, lack of real closures or events, type erasure in generics, and an over obsession with anonymous classes, so I've shied away from doing anything in it.

Still, at least the Java world is trying. While I'd love to just ignore the fact that Java exists, I can't. Lots of interesting projects are done in Java, and a lot of world class tools are written in Java, so from a professional standpoint it'd be good for me to know it.

In the past when I looked into how past Java utilities did "functional" it never felt natural to me. People have suggested [LambdaJ](https://code.google.com/p/lambdaj/) or google's [Guava](https://code.google.com/p/guava-libraries/), but Guava even goes so far to say to _not_ use their functional approach. LambdaJ's is just as verbose as anything else, and in benchmarks its shown to be at least 2 times slower! Not a good sign.

But the coming of Java 8 I think a lot of these problems will be solved. Not to mention I'm sure that projects such as [RxJava](https://github.com/Netflix/RxJava) are ecstatically waiting for this since it will make their (and any other reactive users) lives a whooole lot better.

Anyways, I present to you my first Java program in 5 years:

[java]  
import java.util.List;  
import java.util.concurrent.ExecutionException;

import static java.util.Arrays.asList;

public class Main{  
 public static void main(String[] arsg) throws InterruptedException, ExecutionException {  
 List\<String\> strings = asList("foo", "bar", "baz");

strings.forEach(System.out::println);

ThreadUtil.queueToPool(() -\> {  
 System.out.println("In the damn thread");

return "foo";  
 }).get();

System.out.println("done");  
 }  
}  
[/java]

[java]  
import java.util.concurrent.Callable;  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.Future;

public class ThreadUtil{  
 private static final ExecutorService \_threadPoolExecutor = Executors.newCachedThreadPool();

public static Thread run(Runnable r){  
 Thread thread = new Thread(r);

r.run();

return thread;  
 }

public static Future\<?\> queueToPool(Callable\<?\> r){  
 return \_threadPoolExecutor.submit(r);  
 }  
}  
[/java]

The program does nothing useful. I just wanted to see what it was like to write some java lambda code, and I was pleasantly surprised. Even though lambda's in Java 8 aren't actually first class functions, but are in fact wrappers on interfaces that are tagged with the `@FunctionalInterface` attribute. For example, look at `Supplier` (which is analgous to `Action` in C#, i.e a function with no arguments that when executed returns a value)

[java]  
@FunctionalInterface  
public interface Supplier\<T\> {

/\*\*  
 \* Gets a result.  
 \*  
 \* @return a result  
 \*/  
 T get();  
}  
[/java]

Java's lambda magic looks to rely on the fact that if an interface has the attribute, then it can be auto converted into a lambda as long as there is only one function that is not implemented. You could, however, treat the interface the old fashioned java way: create an anonymous class that implements `get` and execute .get(). Still, why would you want to?

To demonstrate the interface to function mapping you can see both the _new way_ and the _old way_ of doing the same things

[java]  
Supplier\<String\> newWay = () -\> "new way";

Supplier\<String\> oldWay = new Supplier\<String\>() {  
 @Override  
 public String get() {  
 return "old way";  
 }  
};  
[/java]

My only gripe here is that the supplier is still an interface. It's not a first class function, meaning I can't just execute `newWay()`. I have to do `newWay.get()` which seems stupid at first. But, there is a reason.

The reason is that you can now have `default` implementation in interfaces, meaning that you can create an interface instance that has a bunch of stuff defined but create one off lambda overrides of another method. This is pretty neat. Look at this example:

[java]  
@FunctionalInterface  
public interface TestMethods{  
 public void doWork();

default void doOtherWork(){  
 System.out.println("do other work default method");  
 }  
}  
[/java]

Now I can either do the old fashioned way (creating an anonymous class and implementing `doWork`) or I can create an instance that is assigned a lambda, which does the same thing:

[java]  
TestMethods m = () -\> System.out.println("creating the do work method at instantation");

m.doWork();  
m.doOtherWork();  
[/java]

If you have more than one undefined method in a functional interface the compiler will bitch at you, rightfully so.

Anyways, it looks like the Java world is finally growing up!

