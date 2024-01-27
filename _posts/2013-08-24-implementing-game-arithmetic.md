---
layout: post
title: Implementing the game "Arithmetic"
date: 2013-08-24 17:52:55.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- arithmetic
- ast
- expression tree
- F#
- reddit
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560829492;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2020;}i:1;a:1:{s:2:"id";i:4244;}i:2;a:1:{s:2:"id";i:4262;}}}}

permalink: "/2013/08/24/implementing-game-arithmetic/"
---
There is a subreddit on reddit called [/r/dailyprogrammer](http://www.reddit.com/r/dailyprogrammer) and while they don't actually post exercises daily, they do sometimes post neat questions that are fun to solve. About a week ago, they posted [a problem](http://www.reddit.com/r/dailyprogrammer/comments/1k7s7p/081313_challenge_135_easy_arithmetic_equations/) that I solved with F# that I wanted to share. For the impatient, my full source is available at [this fssnip](http://fssnip.net/jy).

The description is as follows:

> Unix[2] , the famous multitasking and multi-user operating system, has several standards that defines Unix commands, system calls, subroutines, files, etc. Specifically within Version 7[3] (though this is included in many other Unix standards), there is a game called "arithmetic". To quote the Man Page[4] :
> 
> Arithmetic types out simple arithmetic problems, and waits for an answer to be typed in. If the answer  
> is correct, it types back "Right!", and a new problem. If the answer is wrong, it replies "What?", and  
> waits for another answer. Every twenty problems, it publishes statistics on correctness and the time  
> required to answer.
> 
> Your goal is to implement this game, with some slight changes, to make this an [Easy]-level challenge. You will only have to use three arithmetic operators (addition, subtraction, multiplication) with four integers. An example equation you are to generate is "2 x 4 + 2 - 5".  
> Author: nint22

## The cheating solution

The cheating solution is to use a dynamic string evaluator. For example, dynamic languages such as ruby, python, and javascript have an `eval` function where you can pass a string to it and it will give you the result of the evaluation. That's no fun and I think that defeats the purpose of the exercise. Someone, somewhere, had to actually write what the eval function does.

However, I did leverage a .NET version of the dynamic evaluation (via the `DataTable` class) to validate my solution in a unit test.

## The F# solution

Whenever I see arbitrary boundaries, I tend to ignore them. My solution works for any size of expression, so unlike some of the other entries that did a lot of specific handling for only four integers, I tested mine up to 1000 terms. The basic principle is my solution generates an expression tree to represent the random expression. To evaluate the expression AST, it evaluates portions of the tree based on a defined set of order of operations. In this way you can partially apply an operation and get a new tree if you want. When evaluating, you have to deal with the fact that some operations have the same weight, such as `+` and `-` so they are evaluated left to right.

## The data types

First, the data types:

```fsharp
  
type Operation =  
 | Mult  
 | Sub  
 | Add  
 override this.ToString() =  
 match this with  
 | Mult -\> "\*"  
 | Sub -\> "-"  
 | Add -\> "+"  
 member this.evaluate =  
 match this with  
 | Mult -\> (\*)  
 | Sub -\> (-)  
 | Add -\> (+)

let orderofOps = [[Mult];[Add;Sub]]  

```

I've created a union type defining the available operations, how to print them out, and what their actual evaluated operation is. The nice thing in F# is that functions are first class. For example, returning `(*)` returns a function of signature `int -> int -> int`.

Also I've defined an order of operations list list. Items in inner lists have the same operation precedence (Add and Sub), and the outer list defines what has to happen first. This way multiplication is evaluated first, then addition and subtraction gets evaluated left to right.

Next is the expression definition. Anyone who's ever worked with syntax tree's should recognize this union pattern:

```fsharp
  
type Expression =  
 | Terminal of int  
 | Expr of Expression \* Operation \* Expression


```

Since it's the idiomatic form of an expression tree.

## Random expressions and numbers

Lets generate some randomness. This defines random numbers and random operations.

```fsharp
  
let rand = new System.Random()

let randNum min max = rand.Next(min, max)

let randomOperation () =  
 match randNum 0 2 with  
 | 0 -\> Mult  
 | 1 -\> Sub  
 | \_ -\> Add  

```

Now I can generate a random expression, where each term is within a min and max range, and the expression is of a passed in length

```fsharp
  
let rec randomExpression min max length =  
 match length with  
 | 0 -\> Terminal(randNum min max)  
 | \_ -\> Expr(Terminal(randNum min max), randomOperation(), randomExpression min max (length - 1))


```

## Display an expression

It'd also be useful to pretty print our expression

```fsharp
  
let rec display = function  
 | Terminal(i) -\> i.ToString()  
 | Expr(left, op, right) -\>  
 String.Format("{0} {1} {2}", display left, op, display right)  

```

This outputs an expression printed like

```
  
8 - 6 \* 8 - 5 \* 9 \* 9  

```

## Tree Evaluation

The last thing we need to do is actually evaluate the tree. Let's break down some of the work into active patterns. I love using active patterns to help hide away complex match statements.

```fsharp
  
let (|TermWithExpression|\_|) predicate expr =  
 match expr with  
 | Expr(Terminal(left), targetOp, Expr(Terminal(right), o, next))  
 when predicate targetOp -\>  
 Expr(Terminal(targetOp.evaluate left right), o, next) |\> Some  
 | \_ -\> None  

```

If the operator passes a predicate and folds the left and right terms into a new terminal if the expression has a left terminal and a right expression. Something of the form:

```fsharp
  
Expr(Terminal(2), Mult, Expr(Terminal(3), Add, Terminal(4)))  

```

The next thing is if we have an expression that is composed of just two terminals.

```fsharp
  
Expr(Terminal(6), Add, Terminal(4))  

```

Again, if the operator passes a predicate we'll fold the two terminals into a new terminal.

```fsharp
  
let (|TermWithTerm|\_|) predicate expr =  
 match expr with  
 | Expr(Terminal(item), targetOp, Terminal(item2))  
 when predicate targetOp -\>  
 Terminal(targetOp.evaluate item item2) |\> Some  
 | \_ -\> None  

```

Finally, lets tie it all into one function

```fsharp
  
let foldExpr expr opsInPrecedence =  
 let rec foldExpr' expr =  
 let shouldEvalOperator o = List.exists (fun i -\> i = o) opsInPrecedence

match expr with  
 | TermWithExpression shouldEvalOperator output -\> foldExpr' output  
 | TermWithTerm shouldEvalOperator output -\> output  
 | Expr(left, o, right) -\> Expr(foldExpr' left, o, foldExpr' right)  
 | Terminal(i) -\> Terminal(i)

foldExpr' expr  

```

## Testing it

I heard a great quote from [Anton Kovalyov](http://anton.kovalyov.net/about/), creator of [JSHint](http://www.jshint.com/), at [QConn NYC](https://qconnewyork.com/) 2013: "_if it's not tested, it's broken_", so here is a test to validate the code:

```fsharp
  
[\<Test\>]  
let arithmeticTest() =

let dt = new DataTable()

for i in [0..100] do  
 let randomExpr = randomExpression 0 10 5

let validationResult = dt.Compute(display randomExpr, "").ToString() |\> Convert.ToInt32

let result = eval randomExpr

printfn "%s = %d = %d" (display randomExpr) validationResult (match result with Terminal(x) -\> x)

result |\> should equal \<| Terminal(validationResult)  

```

This code uses the evaluate capability of the .NET database (found via this [stackoverflow](http://stackoverflow.com/questions/333737/c-sharp-evaluating-string-342-yield-int-18) post) to evaluate the displayed random expression and compare the result to my evaluation of the expression.

## Division

In the original problem description, division was left out to avoid having to deal with divide by zero, but I think that would be pretty easy to handle. The expression folding can know if the right hand term is a zero and the operator is a division, and in that case it can return a `None` solution. So the folding should be modified to return an expression Option type.

