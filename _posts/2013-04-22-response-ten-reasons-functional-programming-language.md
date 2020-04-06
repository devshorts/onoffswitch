---
layout: post
title: A response to "Ten reasons to not use a functional programming language"
date: 2013-04-22 08:00:43.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Rants
tags:
- F#
- functional
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560931038;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4306;}i:1;a:1:{s:2:"id";i:1828;}i:2;a:1:{s:2:"id";i:7777;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/04/22/response-ten-reasons-functional-programming-language/"
---
If you haven't read the [top ten reasons to not use a functional programming language](http://fsharpforfunandprofit.com/posts/ten-reasons-not-to-use-a-functional-programming-language/), I think you should. It's a well written post and ironically debunks a lot of the major trepidations people have with functional languages.

But, I wanted to play devils advocate here. I read a lot of articles on functional and everyone touts a lot of the same reasons to use functional languages, and this post was no different. What these posts always lack, though, is acknowledgement that functional isn't the end all be all of language solutions. It has plenty of problems itself, and it's just as easy to critique it using the same ten reasons. That said, I wanted to address a few of my opinions regarding functional programming using the same format as the original article.

## Reason 1: I don't want to follow the latest fad

The authors point here is that people claim that functional is a fad, but he's right in that it's not. Functional has been around as long as imperative has, in fact [Alonzo Church](http://en.wikipedia.org/wiki/Alonzo_Church) pioneered it with the concepts of [lambda calculus](http://en.wikipedia.org/wiki/Lambda_calculus).

That said, people don't like functional because it's hard to map to their mental model. I don't know about you, but when I think of doing something 4 times I don't think of a recursive loop with an accumulator, I think of a for loop. Making the mental jump to functional isn't easy for everyone which is why it has been slow to adopt to the mainstream.

However, some aspects of functional are reasonably mainstream. Lambdas, options, first class functions and higher order functions are available in many languages such as Ruby, Scala, C#, JavaScript, Python, etc. Some even, like Scala, encourage immutable types! Functional isn't a fad, but pure functional may be.

## Reason 2: I get paid by the line

The point here is that functional languages are usually syntactically shorter. An example the author posts is this:

[csharp]  
public static class SumOfSquaresHelper  
{  
 public static int Square(int i)  
 {  
 return i \* i;  
 }

public static int SumOfSquares(int n)  
 {  
 int sum = 0;  
 for (int i = 1; i \<= n; i++)  
 {  
 sum += Square(i);  
 }  
 return sum;  
 }  
}  
[/csharp]

compared to

[fsharp]  
let square x = x \* x  
let sumOfSquares n = [1..n] |\> List.map square |\> List.sum  
[/fsharp]

But that's cheating. What if we did this instead:

[csharp]  
public int SumOfSquares(int n)  
{  
 return Enumerable.Range(1, n).Select(i =\> i \* i).Sum();  
}  
[/csharp]

And

[fsharp]  
let square x = x \* x  
let sumOfSquares n = [1..n]  
 |\> List.map square  
 |\> List.sum  
[/fsharp]

Now who has more lines? It's all in how you see it. Granted, both are leveraging [higher order functions](http://en.wikipedia.org/wiki/Higher-order_function), but most modern imperative languages support that. Comparing crappy code with good code is never a comparison. Terse code can be written in any (almost) language (sorry Java).

## Reason 3: I love me some curly braces

Personally I don't like space dependent languages since its easy to make scoping mistakes, but that notwithstanding, lets look at some Clojure:

[fsharp]  
(defn run-prep-tasks  
 [{:keys [prep-tasks] :as project}]  
 (doseq [task prep-tasks]  
 (let [[task-name & task-args] (if (vector? task) task [task])  
 task-name (main/lookup-alias task-name project)]  
 (main/apply-task task-name (dissoc project :prep-tasks) task-args)))  
[/fsharp]

While functional usually has less curly braces, many functional languages have a whole lot more parenthesis

## Reason 4: I like to see explicit types

This is a common complaint from people who aren't used to functional, and I can understand, because if someone asked you what the signature below does on first glance what would you say?

[code]  
('State -\> 'T1 -\> 'T2 -\> 'State) -\> 'State -\> 'T1 list -\> 'T2 list -\> 'State  
[/code]

Practiced functional programmers can tell its a function that takes a function (which takes a state, two items, and returns a new state) a seed state, and two lists, and returns a final state. This is the type signature of List.fold2 and its a mouthful!

Compare to the example the author gave:

[csharp]  
public IEnumerable\<IGrouping\<TKey, TSource\>\> GroupBy\<TSource, TKey\>(  
 IEnumerable\<TSource\> source,  
 Func\<TSource, TKey\> keySelector  
 )  
[/csharp]

Immediately at first glance, without caring about the types, you can tell it returns an enumerable, it takes an enumerable, and it takes a function. At the signature you can even see how the source and the selector map to each other. Reading the code you get a sense of how things work together. I won't lie, the signature is nasty, and its verbose. Part of me wishes C# had inferred method signatures, but the other part really likes that I can glance at something and get a big picture overview of what is happening.

On top of that, its easy to make the mistake of passing a function instead of applying a function. Take this example:

[fsharp]  
apply one two  
[/fsharp]

You might think that we are applying two to one, or maybe one to two, or maybe I am passing in a function called one and an argument called two, or maybe both one and two are functions and are being combined and returned as another function, or maybe I meant to curry the apply function by applying one to two like this:

[fsharp]  
apply (one two)  
[/fsharp]

It's very easy to make mistakes like this in functional, especially if the type arguments are generic enough. If the signature for `apply` is a `'a -> 'b -> 'c` then you don't know what you meant! Anyways, this is the complaint people have about implicit vs explicit typing.

## Reason 5: I like to fix bugs

I like to fix type mismatches AND bugs.

## Reason 6: I live in the debugger

I still live in the debugger. To say that a language makes it so that if your code compiles it probably works just boggles my mind. Code can compile fine but be logically completely wrong. This happens in every language! In fact, debugging in fsharp can be complex because of the pipe operator (see my other post on debugging the pipe operator).

## Reason 7: I don't want to think about every little detail

I didn't really get this one. The author talks about how by having all the types matched up you suddenly think of all edge conditions. That's just not true. Like I mentioned above, edge condtions are part of logical flow, not code semantics. You can have all the types match up and still have edge cases you didn't consider.

## Reason 8: I like to check for nulls

This one is fun because I've brought this up with a coworker to discuss before. I, personally, really like the option type, but you can still have nulls. What about

[fsharp]  
let foo = None;

Option.get foo  
[/fsharp]

This results in:

[code]  
System.ArgumentException was unhandled  
 HResult=-2147024809  
 Message=The option value was None  
Parameter name: option  
 Source=FSharp.Core  
 ParamName=option  
 StackTrace:  
 at Microsoft.FSharp.Core.OptionModule.GetValue[T](FSharpOption`1 option)  
 at \<StartupCode$FSharpScratch\>.$Print.main@() in C:\Projects\Program.fs:line 28  
 InnerException:  
[/code]

Oops! So, you still have to match on option discriminated unions for none which means you are still checking for some sort of empty thing. Instead of having

[code]  
if(x != null){  
}  
[/code]

You start having

[code]  
match x with  
 | Some item -\>  
 | \_ -\>  
[/code]

I'm not saying matching is bad, just saying that its wrong to assume you get no exceptions since there are fewer nulls.

A safer design pattern is to use the maybe monad, which can easily be built into every object type in C# using extension methods. I also like the `get` and `getOrElse` pattern that Scala has, meaning that you can either get and it's a None type or get and if it's a None return a default. Much safer.

## Reason 9: I like to use design patterns everywhere

Design patterns apply to any code of any language. To just write flat code with no organization or thought to structure is going to break large apps. Those patterns exist, I'm sure, even in functional applications. And if they don't, I'd be skeptical of their extensiblity and robustness.

You always want to segment out 3rd party libraries behind proxies, you want to hide how things are created with factories, and you want to make sure that you interface with abstract classes and interfaces when necessary so you can inject different implementations of things. In fact, the F# team have videos showing how to do certain [design patterns](http://channel9.msdn.com/posts/Tao-Liu-F-Design-Patterns) with F#!

## Reason 10: It's too mathematical

This is something I've never heard people mention when complaining about functional, but maybe it's true. I don't know. I can't comment on this one.

## Conclusion

I think the article is hilarious and well written, and I am a huge proponent of functional languages. But I find that some things about functional do annoy me. One of the reasons I really like f# is because you can do imperative work when you need to. That said, I think the big language winners will be languages like C# and Scala that embrace functional paradigms but also let you build with imperative.

