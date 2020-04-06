---
layout: post
title: Capturing mutables in f#
date: 2013-04-08 08:00:26.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- closures
- F#
- heap
- stack
meta:
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _edit_last: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560183675;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1873;}i:1;a:1:{s:2:"id";i:3723;}i:2;a:1:{s:2:"id";i:4028;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/04/08/capturing-mutables-f/"
---
I was talking about F# with a coworker recently and we were discussing the merits of a stateless system. Both of us really like the enforcement of having to inject state, and when necessary, returning a new modified copy of state. Functional languages want you to work with this pattern, but like with all things software, it's good to be able to break the rules. This is one of the things I like about F#, you can create mutables and do work imperatively if you need to.

But, there is a small caveat with mutables: you can't close over them. Look at the following example:

[fsharp]  
let g() =  
 let mutable f = 0;

fun () -\> Console.WriteLine f  
[/fsharp]

The intent is that calling `g()` would give you a new function that writes `f` to the console. In C# it would be the same as

[csharp]  
public Action g(){  
 int f = 0;  
 return () =\> Console.WriteLine(f);  
}  
[/csharp]

Both examples look functionally the same, but the F# example actually gives you the following compiler error:

> The mutable variable 'f' is used in an invalid way. Mutable variables cannot be captured by closures. Consider eliminating this use of mutation or using a heap-allocated mutable reference cell via 'ref' and '!'.

But, the C# version is totally fine. Why?

The reason is because F# mutable values are [always stack allocated](http://stackoverflow.com/a/4004715/310196). To close on a variable, the variable needs to be allocated on the heap (or copied by value). This is why you can close on objects that aren't mutable (you close on their reference) and values that aren't mutable (they are closed by value, i.e. copied). If you closed on a stack allocated type it wouldn't work; stack objects are popped off after the function loses scope. This is the basis of [stack unwinding](http://en.wikipedia.org/wiki/Call_stack#Unwinding). After the stack is unwound, the reference to the value you closed on would point to garbage!

So why does the C# version work? `f` looks like a stack allocated value type to me. The nuance is that the C# compiler actually makes `f` become a heap allocated value type. Here is a quote from [Eric Lipperts](http://blogs.msdn.com/b/ericlippert/archive/2010/09/30/the-truth-about-value-types.aspx) blog explaining this (emphasis mine):

> in the Microsoft implementation of C# on the desktop CLR, value types are stored on the stack when the value is a local variable or temporary that **is not a closed-over local variable of a lambda or anonymous method** , and the method body is not an iterator block, and the jitter chooses to not enregister the value.

So C# actually moves the value type to the heap to be declared because it needs to be accessed later via the closure. If you didn't do that, then the value type wouldn't exist when the closure is executed since the stack reference would have been lost (stacks are popped off when functions return).

F#, then, is much stricter about its stack vs heap allocations and opted to not do this magic for you. I think their decision aligns with the functional philosophy of statelessness; they obviously could have done the magic for you but chose not to.

Instead, if you do need to return a captured mutable value in a function closure you have to use what is called a [reference cell](http://msdn.microsoft.com/en-us/library/dd233186.aspx). All a reference cell is is a heap allocated mutable variable, which is exactly what you need for returned closures to work.

A modified version of our example that would now work looks like this:

[fsharp]  
let g() =  
 let f = ref 0;

fun () -\> Console.WriteLine !f

g()()  
[/fsharp]

Notice the `!` which dereferences the cell. This example outputs

[code]  
0  
[/code]

Without the `!`, though, you'd get

[code]  
Microsoft.FSharp.Core.FSharpRef`1[System.Int32]  
[/code]

Showing you that `f` isn't really an int, it's a boxed heap value of an int.

