---
layout: post
title: Tracing computation expressions
date: 2013-01-18 15:43:48.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- F#
- monads
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '979357360'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"d1e23d6ffe5a1de5892bb68020be156f";a:2:{s:7:"expires";i:1555043233;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:4463;}i:2;a:1:{s:2:"id";i:4226;}}}}

permalink: "/2013/01/18/tracing-computation-expressions/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/tracing-computation-expressions/)_

F# has a novel syntax feature called [_computation expressions,_](http://en.wikibooks.org/wiki/F_Sharp_Programming/Computation_Expressions) which lets you build complex monadic expressions with minimal syntax. Commonly shied away from, a [monad](http://stackoverflow.com/questions/2704652/monad-in-plain-english-for-the-oop-programmer-with-no-fp-background/2704795#2704795) is simple: it's a function whose input is some state. A monad usually manipulates the state and returns a new state (which can be handed off to another monad).

Monad's are maybe best known from [Haskell](http://www.haskell.org/haskellwiki/Monad), but they exist in scheme, ML, clojure, scala, and even show up in [C#](http://devtalk.net/csharp/chained-null-checks-and-the-maybe-monad/) and other imperative languages.

While the computation expression syntax is cool, short of the [maybe monad](http://en.wikipedia.org/wiki/Monad_(functional_programming)#The_Maybe_monad), and the baked-in [async keyword](http://msdn.microsoft.com/en-us/library/dd233250.aspx), I wasn't sure what I could do with this. Thankfully, I found an interesting post by [Luis Diego Fallas](http://langexplr.blogspot.com/2008/10/using-f-computation-expressions-to-read.html) who posted an F# code sample leveraging computation expressions to read binary formatted files. However, if you are like me, and trying to better understand the power of computation expressions, tracing through these samples can be difficult. It's not because they are poorly written, but mostly because they are jumping into more complex usages.

To clarify what's going on with computation expressions, I wanted to show what is passing through the monad. Computation expressions (also known as monads or workflows) can be tricky because you are overriding the language syntax. So what is a "let!" statement in one workflow is not the same as in another workflow. On top of that statements themselves can return a value, not just have a left hand side assignment. If you feel your brain start to hurt, that's OK. It will get better.

## An example

Let's start with a small sample to see how information passes through the workflow. This makes it easier to understand what computation expressions are doing:

[fsharp]  
open System

type State =  
 | Current of int \* State  
 | Terminated

type Builder() =  
 member this.Bind(value:int, rest:int-\>State) =  
 State.Current((value, rest(value)))

member this.Return(returnValue:int) = State.Current(returnValue, State.Terminated)

let builder = new Builder()

let build \_ = builder{  
 let! x = 1  
 let! y = 2  
 return 3  
 }

let stateChain = build()

let rec formatState (chain:State) =  
 match chain with  
 | State.Terminated -\> Console.WriteLine()  
 | State.Current(i, next) -\> Console.WriteLine("State: {0}", i)  
 formatState next

formatState stateChain

Console.ReadKey() |\> ignore  
[/fsharp]

Which prints out

[csharp]  
State: 1  
State: 2  
State: 3  
[/csharp]

## Desugaring

Before we trace through what's happening, lets desugar the expression. If you aren't familiar with the bang syntax, a let! statement will compile into an execution on the builders Bind function and a return will compile into an execution on the builders Return function. Let's take the original expression:

[fsharp]  
let build \_ = builder{  
 let! x = 1  
 let! y = 2  
 return 3  
 }  
[/fsharp]

And show the same thing but without the syntactic sugar.

[fsharp]  
let desugared =  
 builder.Bind(1, fun next -\>  
 let x = next // next = 1 since we took the input value of the bind,  
 // and passed it to the next lambda

let returnedState =  
 builder.Bind(2, fun next2 -\>  
 let y = next2 // next2 is the value 2 here

let terminatedState = builder.Return(3)

//terminated state is now  
 //State.Current(3, State.Terminated)

terminatedState  
 )

// the state here is  
 // State.Current(2, State.Current(3, State.Terminated))  
 returnedState  
 )  
[/fsharp]

The important thing to understand here is how the let! statement is deconstructed. The right hand side is the `value` input to the bind. Everything below the let! statement (and including the left hand side of the assignment), is inside of the bind lambda (passed to the `rest` argument of the bind function). The first thing the lambda does is actually apply the left hand side assignment with the input from the bind. In this case, the first let! statement assigns `x = 1`. Then it executes the remaining function, being the other let! and return statements.

## Tracing it through

In the original sample, I've defined a discriminated union called `State` representing the current state. This union has two types. One, called `Current`, contains an integer as well as a link to the next state in the form of a tuple. The other, `Terminated`, is a value that we can use as a terminator for our state link. The example is really just for demonstration, since by being able to capture the state it's easier to understand how computation expressions work; it gives us a sense of where the monad has been.

Let's take this one step at a time, with an even simpler example based on the above code. It's important to understand the deconstruction. The compiler will translate our computation expressions into invocations on the builder.

[![builderDeconstruction1](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-1-300x227.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-1-e1356539394542.jpg)

To desguar it we take the left hand side and everything after

[![builderDeconstruction2](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-2-300x186.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-2-e1356539595127.jpg)

And move it to a lambda. This lambda is what is going to be passed as the `rest` argument to the builder's bind function

[![builderDeconstruction3](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-3-300x167.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-3-e1356539579118.jpg)

The right hand side is going to be applied to the `value` argument of the bind function

[![builderDeconstruction4](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-4-300x204.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-4-e1356539561372.jpg)

Go back and look at how we've defined the bind function:

[fsharp]  
member this.Bind(value:int, rest:int-\>State) =  
 State.Current((value, rest(value)))  
[/fsharp]

Here the `value` argument is 1. The second argument, `rest`, is the lambda. The lambda is going to have to return a `State` union since `State.Current` expects an integer, State tuple.

When we execute the lambda inside the bind we pass the value (1) to the lambda, but we've also captured the current value. This means that this bind is going to return:

[fsharp]  
State.Current(1, rest(1))  
[/fsharp]

So here we apply the value to the function

[![builderDeconstruction5](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-5-300x195.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo-5-e1356539528361.jpg)

This is where the left hand side statement (x) now gets assigned.

[![builderDeconstruction6](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo6-300x184.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/12/photo6-e1356539509430.jpg)

Now what about

[fsharp]  
builder.Return(2)  
[/fsharp]

Remember we defined the return function to return

[fsharp]  
member this.Return(returnValue:int) = State.Current(returnValue, State.Terminated)  
[/fsharp]

So with our simplified example this will return

[fsharp]  
State.Current(2, State.Terminated)  
[/fsharp]

The previous lambda now returns that same value, since that's the last line of the statement. So we're back now to the original bind function:

[fsharp]  
member this.Bind(value:int, rest:int-\>State) =  
 State.Current((1, rest(1)))  
[/fsharp]

But `rest(1)` returns `State.Current(2, State.Terminated)`. Our final builders return value, in this example, is

[fsharp]  
State.Current(1, State.Current(2, State.Terminated))  
[/fsharp]

All the computation builder syntax is doing is just sugaring our statements up to give us these broken up functions.

Back to the original sample. We added a second `let!` statement in there:

[fsharp]  
let build \_ = builder{  
 let! x = 1  
 let! y = 2  
 return 3  
 }  
[/fsharp]

Now, hopefully, you should be able to see how the final return from the computation expression is a state object representing what happened in the monad:

[fsharp]  
State.Current(1, State.Current(2, State.Current(3, State.Terminated)))  
[/fsharp]

## Combine and Yield

Computation expressions have more than just let! and return statements though. Once you get used to tracing through and thinking about the computation builder, it becomes easier to start writing workflows. Just for kicks, I wanted to see if I could write a computation expression to evaluate a basic arithmetic expression. Here I'm using [partial functions](http://en.wikipedia.org/wiki/Currying) and the `Combine` property of the builder to build out the expression. If you wanted to, you can even use computation expressions within computation expressions. There's nothing keeping you from doing that.

In general, there is a bunch of reserved syntax that maps to specific builder functions. The [msdn](http://msdn.microsoft.com/en-us/library/dd233182.aspx) on computation syntax has these all defined.

[fsharp]  
open System

type BuildTest() =  
 member this.Combine(currentStatement, value) = currentStatement(value)  
 member this.Return(value) = value  
 member this.Yield(item) = item  
 member this.Delay(item) = item()

let builder = new BuildTest()

type math() =  
 member this.add x y = x + y  
 member this.mult x y = x \* y

let m = new math()

let build \_ = builder{  
 yield m.mult 2 // 2 \* (1 + (2 + (3 + 0))  
 yield m.add 1 // 1 + (2 + (3 + 0)  
 yield m.add 2 // 2 + (3 + 0)  
 yield m.add 3 // (3 + 0)  
 return 0 // 0  
 }

let run = build()

let monader = printf "%s" ("got " + run.ToString())

Console.ReadKey() |\> ignore  
[/fsharp]

This snippet evaluates to 12.

You can even add precedence by evaluating a computation expression within the computation expressions

[fsharp]  
builder{  
 yield m.mult 2 // 2 \* (1 + (8 + 0))  
 yield m.add 1 // 1 + (8 + 0)

let parenth = builder{  
 yield m.mult 4 // 4 \* 2  
 return 2 // 2  
 }

yield m.add parenth // 8 + 0

return 0 // 0  
}  
[/fsharp]

Which evaluates to 18.

## Tracing Combine and Yield

Just like before, there's a bunch of magic going on here, so it's easier if you follow along with the desugared version of the original arithmetic expression below.

[fsharp]  
let desugared = builder.Delay(  
 fun () -\> builder.Combine(builder.Yield(m.mult 2),  
 builder.Delay(  
 fun() -\> builder.Combine(builder.Yield(m.add 1),  
 builder.Delay(  
 fun() -\> builder.Combine(builder.Yield(m.add 2),  
 builder.Delay(  
 fun() -\> builder.Combine(builder.Yield(m.add 3),  
 builder.Delay(  
 fun() -\> builder.Return(0))))))))))  
[/fsharp]

Each monadic function is wrapped in a `Delay`, which promptly executes it. Look at the builder's delay declaration - it takes a function and executes it.

In our builder, the `Yield` just returns the same value. It doesn't do much but we needed to implement it to use the computation expression syntax.

What we pass to the delay is an anonymous function that has a combine statement. `Combine`s take two things and produce a third. Here, we are passing the current partial function as the first argument (via the yield), and the value we want to use to evaluate this partial function as the second argument. However, the second argument isn't actually evaluated till the end. The combine will then apply the second argument (an integer) to the first argument (a partial function that takes an integer).

For the basic arithmetic example, the final delay function returns 0. You can think of this as a "seed." If you think of it like a fold operation, this is very similar. When we finally return the seed, we bubble each evaluated expression back up the stack (starting with 0), so read the desugared version from the bottom up. In this way, we are executing the current curried statement with the previous statements evaluated value in the `Combine` method of the builder. Not the most practical application, but I thought it was a fun exercise.

If you are confused why this example's desguaring contains the Delay method and the original example didn't, it's because the sugaring happens differently depending which builder constructs you use.

## Under the hood

When you [decompile](http://www.jetbrains.com/decompiler/) a computation expression, each monadic function gets compiled into it's own class representing a monad. In our arithmetic operation example, this is a Combine, Yield, Delay trio. It's not easy to read since the function names have been mangled, but you can see the general pattern here (formatting added).

[csharp]  
[Serializable]  
internal class runu004089 : FSharpFunc\<Unit, int\>  
{  
 internal runu004089()  
 {  
 }

public override int Invoke(Unit unitVar0)  
 {  
 return ExpresionsTest.builder.Combine\<int, int\>(  
 ExpresionsTest.builder.Yield\<FSharpFunc\<int, int\>\>(  
 (FSharpFunc\<int, int\>) new ExpresionsTest.runu004089u002D1(2, ExpresionsTest.m)),  
 ExpresionsTest.builder.Delay\<int\>(  
 (FSharpFunc\<Unit, int\>) new ExpresionsTest.runu004091u002D2()));  
 }  
}  
[/csharp]

This decompliation represents the following sub-block.

[fsharp]  
builder.Combine(  
 builder.Yield(m.add 2), builder.Delay( (\*next function\*) )  
)  
[/fsharp]

Notice in the decompiled block that the class name is `runu004089` and the executable expression is compiled into an `Invoke` that returns an int. The decompiled assembly will actually be littered with these classes with mangled names, following a naming format of the target variable name (`run`) and an identifier (`u004089`). You can always decompile the computation expression to get a sense for how it's been desugared.

## Conclusion

I said in the beginning that monads are simple, but I'll admit that I lied. Monads are tricky, there's no denying that. Maybe that's why there is [no shortage](https://www.google.com/search?q=%22what+is+a+monad%22) of blog posts trying to explain the monad over and over again. But, in the end, once you wrap your head around it, I think computation expression syntax is a cool way of using the concept of a monad by decoupling what something is defined to do, vs how it's actually executed.

I highly suggest running the examples and actually stepping through them bit by bit if you are having trouble following what is happening. Being able to see a desugared version of the code and using a debugger to inspect locals while stepping through examples makes it a lot clearer to see whats happening.

## More reading

If you are curious here are some links to further reading [explaining monads](http://stackoverflow.com/questions/44965/what-is-a-monad) and computation expressions (such as the [F# wikibook](http://en.wikibooks.org/wiki/F_Sharp_Programming)). [Don Syme](http://blogs.msdn.com/b/dsyme/archive/2007/09/22/some-details-on-f-computation-expressions-aka-monadic-or-workflow-syntax.aspx) also has a few posts explaining things really well that are definitely worth checking out.

