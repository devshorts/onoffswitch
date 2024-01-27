---
layout: post
title: Debugging piped operations in F#
date: 2012-12-14 09:34:27.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- Debugging
- F#
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '963156174'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559527959;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:3565;}i:2;a:1:{s:2:"id";i:4197;}}}}

permalink: "/2012/12/14/debugging-piped-sequences-f/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/debugging-piped-sequences-f/)_

# A little on the pipe operator

In F# you can create [piped operations](http://www.c-sharpcorner.com/uploadfile/rmcochran/fsharp-types-and-the-forward-pipe-operator/) using the `|>` operator. This takes the output of the previous statement and funnels it as the input to the next statement. Using the pipe operator, a statement like this:

```fsharp
  
x |\> f |\> g |\> h  

```

Means having functions nested like this:

```fsharp
  
h(g(f(x))  

```

So a piece of code like this:

```fsharp
  
let print item = Console.WriteLine(item.ToString)

let seqDebug =  
 [0..1000]  
 |\> List.map (fun i -\> i + 1)  
 |\> List.filter (fun i -\> i \< 5)  
 |\> List.head  
 |\> print  

```

Decompiles into this (formatting added):

```csharp
  
[DebuggerBrowsable(DebuggerBrowsableState.Never)]  
internal static Unit seqDebugu00407;

public static void mainu0040()  
{  
 Program.print(  
 ListModule.Head(  
 ListModule.Filter((FSharpFunc\<int, bool\>) new Program.seqDebugu004010(),  
 ListModule.Map\<int, int\>((FSharpFunc\<int, int\>) new Program.seqDebugu00409u002D1(),  
 SeqModule.ToList(Operators.CreateSequence(  
 Operators.OperatorIntrinsics.RangeInt32(0, 1, 1000)))))));

u0024Program.seqDebugu00407 = (Unit) null;  
}  

```

Which really boils down to:

```csharp
  
seqDebug = Print(Head(Filter(Map(sequence))))  

```

The F# syntax is nice because it lets us write code from the outside in, instead of inside out.

# Debugging it

Now that we know what F# is doing, lets say we want to debug the print statement. You can't use your normal "[Step Over](http://msdn.microsoft.com/en-us/library/ek13f001.aspx)" F10 key to go through your piped statement here because it compiles down to a one line group of nested functions. We could use the "Step Into" key (F11) to step into the entire sequence but then we have to execute the anonymous map lambda 1001 times just to get to the next statement. Then another 1001 for the filter. Then the head statement, and finally, our print. No thanks.

Thankfully, Visual Studio has thought of this and you can use the _[Step Into Specific](http://msdn.microsoft.com/en-us/library/7ad07721.aspx)_ functionality. This lets you see the list of nested functions at that line and you can jump into whatever you need to here. _Step Into Specific_ isn't an F# only feature, but I never realized it existed until I ran into this scenario.

[![Step into Specific](http://tech.blinemedical.com/wp-content/uploads/2012/12/stepIntoSpecific2-300x105.png)](http://tech.blinemedical.com/wp-content/uploads/2012/12/stepIntoSpecific2.png)

The example is a little trivial, since you would've just put a breakpoint in the print statement, right? But what if you are piping through F# operators like `List.map` and `List.filter`? In these cases it can be hard to know what is the direct input to these functions since the input argument is automatically applied. For these scenarios, a simple identity function can be really helpful:

```fsharp
  
let identity item = item

let seqDebug =  
 [0..1000]  
 |\> List.map (fun i -\> i + 1)  
 |\> identity  
 |\> List.filter (fun i -\> i \< 5)  
 |\> List.head  

```

So you can sprinkle in your identity function and put breakpoints there. This way you can inject yourself into the middle of this sequence.

# Piping with functions that return unit

Taking this one step further, sometimes I want to print out a value in the middle of the sequence, or call a function that has a return type of `unit` but continue piping. Because let's be honest here, when all else fails nothing beats a well placed `printf` in your code. But, we're left with a small dilemma: since pipes take the output of the last function and use it as the input to the next function we can't really use print statements. Both `printf` and `Console.WriteLine` effectively return a `void`. Putting them in the middle of a chain won't work since their output won't map to the next functions input (unless that next function takes `unit`).

However, F# lets you define [your own operators](http://msdn.microsoft.com/en-us/library/dd233204.aspx), so I created one that I like to call the "argument identity" that executes a function which returns void and then returns the original argument (acting as an argument identity function):

```fsharp
  
let (~~) (func:'a-\> unit) (arg:'a) = (func arg) |\> fun () -\> arg  

```

The `~~` symbol is a [prefix](http://msdn.microsoft.com/en-us/library/dd233204.aspx) operator that takes a function of one argument that returns unit, then closes the argument into a function with type `unit -> 'a`. Then I pipe the return value (unit) to the anonymous function (that takes unit) which will return the closed value of the original argument. Now I can do things like this:

```fsharp
  
let (~~) (func:'a-\> unit) (arg:'a) = (func arg) |\> fun () -\> arg

let seqDebug =  
 [0..1000]  
 |\> List.map (fun i -\> i + 1)  
 |\> ~~ Console.WriteLine  
 |\> List.filter (fun i -\> i \< 3)  
 |\> ~~ Console.WriteLine  
 |\> List.head  
 |\> ~~ Console.WriteLine  

```

Which prints out

```csharp
  
[1; 2; 3; ...]  
[1; 2]  
1  

```

You obviously don't need your own operator, you can make it a named helper function if you want. Either way, some sort of argument identity function is useful in these scenarios.

# Disassemble the pipe

And of course, when all else fails, you can break up the sequence into a series of `let` statements to debug it the old fashioned way.

```fsharp
  
let seqDebugDecomposed =  
 let source = [0..1000]  
 let sourcePlusOne = List.map (fun i -\> i + 1) source  
 let filteredSource = List.filter (fun i -\> i \< 3) sourcePlusOne  
 let listHead = List.head filteredSource  
 print listHead  

```

