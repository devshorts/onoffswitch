---
layout: post
title: The Arrow operator
date: 2013-10-14 08:00:20.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- Arrows
- F#
- haskell
meta:
  _wpas_done_all: '1'
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561173373;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4725;}i:1;a:1:{s:2:"id";i:4320;}i:2;a:1:{s:2:"id";i:4348;}}}}

permalink: "/2013/10/14/arrow-operator/"
---
Continuing my journey in functional programming, I decided to try doing the [99 haskell problems](http://www.haskell.org/haskellwiki/99_questions/) to wean my way into haskell. I've found this to be a lot of fun since they give you the answers to each problem and, even though I have functional experience, the _haskell_ way is sometimes very different from what I would have expected.

For example, I discovered [Arrows](http://en.wikipedia.org/wiki/Arrow_(computer_science)) via the following problem:

> Run-length encoding of a list. Implement the so-called run-length encoding data compression method. Consecutive duplicates of elements are encoded as lists (N E) where N is the number of duplicates of the element E.
> 
> Example:
> 
> \* (encode '(a a a a b c c a a d e e e e))  
> ((4 A) (1 B) (2 C) (2 A) (1 D)(4 E))  
> Example in Haskell:
> 
> encode "aaaabccaadeeee"  
> [(4,'a'),(1,'b'),(2,'c'),(2,'a'),(1,'d'),(4,'e')]

My initial solution I did the way I'd probably write it in F#:

[haskell]  
encode :: Eq a =\> [a] -\> [(Int, a)]  
encode a = map (\x -\> (length x, head x)) . group $ a  
[/haskell]

But, one of the alternate solutions to the problem was cleaner and used an operator I'd never seen

[haskell]  
encode :: Eq a =\> [a] -\> [(Int, a)]  
encode a = map (length &&& head) . group $ a  
[/haskell]

What the hell was `&&&`? If we break down the function `length &&& head` a little we'll see it has a signature of

[haskell]  
(length &&& head) :: [a] -\> (Int, a)  
[/haskell]

The function takes a list and will apply the first function to the list as the first element of the tuple (length) and then apply the second function to the list and that'll give you the second element of the tuple.

Turns out this is part of the `Control.Arrow` module and defines an interesting way to combine logical steps using tuples and several basic transformation functions. There is a great intro to arrows on the [haskell wiki](http://www.haskell.org/haskellwiki/Arrow_tutorial) which I followed and ended up simulating in F#.

## Translating to F#

A quick translation attempt turned out the following small module

[fsharp]  
module Arrow =  
 let split x = (x, x)  
 let combine f (x, y) = f x y  
 let first f (a, b) = (f a, b)  
 let second f (a, b) = (a, f b)  
 let onTuple f g = first f \>\> second g  
 let onSingle f g = split \>\> (onTuple f g)  
 let (.\*\*\*.) = onTuple  
 let (.&&&.) = onSingle  
 let onSingleCombine op f g = (onSingle f g) \>\> combine op  
[/fsharp]

To break down what's going on here, we can follow an example posted in the haskell wiki.

[fsharp]  
let div2 x = x / 2  
let m3p1 x = 3\*x + 1

let example = onSingleCombine (+) div2 m3p1 8  
[/fsharp]

What this is really doing is:

- Split 8 into (8, 8)
- Apply the first function to the first element (`div2 8`) with a resulting tuple of (4, 8)
- Apply the second function to the second element (`m3p1 8`) with a resulting tuple of (4, 25)
- Apply the combiner `(+)` to the resulting tuple which gives the answer of `29`

While I think the example is a little contrived, it's really cool how you can leverage arrows to do this kind of sequencing work. I'll certainly find use for this!

