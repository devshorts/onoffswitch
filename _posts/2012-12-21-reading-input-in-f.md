---
layout: post
title: Reading input in F#
date: 2012-12-21 15:35:59.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags: []
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '986440027'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559832280;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3565;}i:1;a:1:{s:2:"id";i:4365;}i:2;a:1:{s:2:"id";i:4028;}}}}

permalink: "/2012/12/21/reading-input-in-f/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/reading-input-in-f/)_

I've been playing with F# lately, much to the the chagrin of [Sam](http://tech.blinemedical.com/author/samuel-neff/), but I still think it's fun as an excersize in thinking differently. I also find its terseness lets you prototype ideas quickly, encouraging you to experiment and tweak your code. However, I'm much more used to imperative programming, so when I started writing an F# program that needed user input I hit a small roadblock: how do I get input, validate, and ask for it again if I want to stay purely functional and leverage immutable values?

In an imperative language you might write something like this:

[csharp]  
String path = null;  
while(true)  
{  
 path = Console.ReadLine();  
 if (File.Exists(path))  
 {  
 break;  
 }

Console.WriteLine("File doesn't exist");  
}  
[/csharp]

You can't declare `path` inside the while loop or its loses its scope. If you need to use `path` outside of the while loop, then it might seem like you have to let path be mutable. But, what if we did this:

[csharp]  
private String GetPath()  
{  
 while(true)  
 {  
 var path = Console.ReadLine();  
 if (File.Exists(path))  
 {  
 return path;  
 }  
 Console.WriteLine("File doesn't exist");  
 }  
}  
[/csharp]

Now we don't ever update any variables. We only ever use direct assignment. This sounds pretty functional to me. But, we still can't directly translate into F#. Remembering that in F# the last statement is the return value, what does this return?

[csharp]  
let falseItem =  
 while true do  
 false  
[/csharp]

This is actually an infinite loop; the while loop won't ever return `false`. In F#, a while loop can't return from it's body, since the body expression return type [has to be](http://msdn.microsoft.com/en-us/library/dd233208.aspx) of type unit. If you imagine the while loop as a function that takes a predicate and a lambda for the body then this makes sense. The `whileLoop` function will execute the body as long as the predicate returns true. So, in psuedocode, it kind of looks like this

[csharp]  
whileLoop(predicate, body) = {  
 while predicate() do {  
 body()  
 }  
}  
[/csharp]

Now what? Well, turning this while loop into a recursive structure with immutable types is actually pretty easy:

[csharp]  
let rec documentPath =  
 fun () -\>  
 Console.Write("File path: ")  
 let path = Console.ReadLine()  
 if not(File.Exists path) then  
 Console.WriteLine("File does not exist")  
 documentPath()  
 else path  
[/csharp]

The trick here is to define `documentPath` as a recursive function. Either the function returns a valid path, or it calls itself executing the next "step" in our while loop. Also, since we don't need to do any work after the recursive function call, F# can optimize this to use [tail call optimization](http://stackoverflow.com/questions/310974/what-is-tail-call-optimization). The `documentPath` variable is of type `unit -> string` meaning it's a function that takes a unit type and returns a string. To actually get the path, we execute `documentPath()`, where `()` is the unit type.

Now we have a function that uses immutable types, but continuously reads in user input and won't return until the input is valid.

Though, if you really want to use imperative style loop breaks, [you can](http://tomasp.net/blog/imperative-i-return.aspx), but it's not trivial.

