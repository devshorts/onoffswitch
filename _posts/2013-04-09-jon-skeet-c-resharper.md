---
layout: post
title: Jon Skeet, C#, and Resharper
date: 2013-04-09 19:21:04.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rants
tags:
- c#
- resharper
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1556356876;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4493;}i:1;a:1:{s:2:"id";i:3656;}i:2;a:1:{s:2:"id";i:3565;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/04/09/jon-skeet-c-resharper/"
---
Today, at 1pm EST, the venerable [Jon Skeet](http://stackoverflow.com/users/22656/jon-skeet) had a goto meeting webinar sponsored by [JetBrains](http://www.jetbrains.com/) reviewing weird and cool stuff about C# and Resharper. For those who aren't in the know, [Resharper](http://www.jetbrains.com/resharper/) is a static analysis tool for C# that is pretty much the best thing ever. Skeet's a great speaker and my entire team at work and I watched the webinar in our conference room while eating lunch.

I took some notes and wanted to share some of the interesting things that Jon mentioned. You can watch the video [here](http://blogs.jetbrains.com/dotnet/2013/04/webinar-recording-jon-skeet-inspects-resharper/). It's an hour long and definitely worth viewing.

## Recursive Parameterization

Skeet talked about how Resharper, and in fact the C# compiler lets you do weird stuff like this:

[csharp]  
public class SuperContainer\<T\>  
{

}

public class Container\<T\> : SuperContainer\<Container\<Container\<T\>\>\>  
{  
}  
[/csharp]

Even though this leads itself to recursive parameterization. Compiling this is just fine though. However, even if its not used in an assembly, if you run unit tests for that assembly you'll get:

![recursiveParameterization.](http://onoffswitch.net/wp-content/uploads/2013/04/recursiveParameterization.-600x87.png)

This is because unit tests usually use reflection to test your assemblies. If you don't use a unit test, and you never access it you won't have an issue. The problem, Skeet told me, isn't in the C# compiler, it's that the CLR goes, as Skeet put it, "_bang_"

## Access to modified closures

Jon talked about the problem of accessing modified closures and how it's different in C#5 vs previous versions. The problem is described like this:

[csharp]  
var list = new List\<Action\>();  
foreach (var i in Enumerable.Range(0, 10))  
{  
 list.Add(() =\> Console.WriteLine(i));  
}  
[/csharp]

In C# 4, the variable `i` is the same reference for each iteration. This means that when you capture the value in a lambda, you are closing on its reference. Running this, you are going to get

[code]  
9  
9  
9  
9  
9  
9  
9  
9  
9  
9  
[/code]

The C# 4 and earlier solution is to make sure that a new variable is created each time the iteration runs:

[csharp]  
var list = new List\<Action\>();  
foreach (var i in Enumerable.Range(0, 10))  
{  
 int tmp = i;  
 list.Add(() =\> Console.WriteLine(tmp));  
}  
[/csharp]

This gives you the right answer. But in C# 5 they changed the handling of foreach internally to give you the expected behavior: you will close on different references each time.

## Covariance

Jon then spent a short bit discussing covariance between objects and how you can induce runtime failures, but resharper doesn't warn you about it. For example, the following code is compilable, but not runnable:

[csharp]  
string[] x = new string[10];  
object[] o = x;  
o[0] = 5; // breaks  
[/csharp]

## Statics in generic types

The next thing Jon talked about was the Resharper warning when you have a static member variable as part of a class with generics. For example:

[csharp]  
public class Foo\<T\>  
{  
 public static string Item { get; set; }  
}

[Test]  
public void StaticTest()  
{  
 Foo\<String\>.Item = "a";  
 Console.WriteLine(Foo\<String\>.Item);

Foo\<int\>.Item = "b";  
 Console.WriteLine(Foo\<String\>.Item);  
 Console.WriteLine(Foo\<int\>.Item);  
}  
[/csharp]

Which prints out

[code]  
a  
a  
b  
[/code]

Interestingly enough, Resharper 7 gives me no warning on using a static item in a templated class. The problem is really when you think you have a cache or some other static item per class, but its created once per **type**. This was new info to me so I thought this was pretty cool.

## Virtual method call in constructor

Jon's mentioned it on twitter before, and it was cool to see him mention it in his webinar, but you can get into very strange things when you call a virtual method from a base constructor. For example:

[csharp]  
public class Base  
{  
 protected int item;

protected Base()  
 {  
 VirtualFunc();  
 }

public virtual void VirtualFunc()  
 {  
 Console.WriteLine(item);  
 }

}

public class Derived : Base  
{  
 public Derived()  
 {  
 item = 1;

VirtualFunc();  
 }

public override void VirtualFunc()  
 {  
 if (item != 1)  
 {  
 throw new Exception("Should never do this");  
 }  
 }  
}  
[/csharp]

Which prints out

[code]  
System.Exception : Should never do this  
[/code]

Basically the base class constructor is called first, so you haven't set the member field in the derived constructor. This means that if you run into this problem you have no way of assuring that items are initialized, even if they may be set in the constructor. Resharper, thankfully, gives you a warning about this. So follow it's advice!

## Miscellaneous c# weirdos

Skeet ended with a spattering of random C# weirdness, like being able to declare a class called `var` even though `var` is a keyword. Also, comparing of doubles can be...well, odd:

[csharp]  
[Test]  
public void CompareDouble()  
{  
 Console.WriteLine(double.NaN == double.NaN);

var x = double.NaN;

Console.WriteLine(x == x);  
}  
[/csharp]

Here, Resharper says "hey, just change these values to true, they're always going to be true", but actually this prints out

[code]  
False  
False  
[/code]

What?

## Conclusion

Jon quoted an unnamed source that describes the content of the webinar:

> You are entering dark places

And I tend to agree. Thanks for the great presentation Jon and the JetBrains team.

EDIT:

Skeet tweeted his [sample solution project](https://github.com/hhariri/Tidbits) that he used in the webinar. For more samples of C# weird/cool stuff check it out!

