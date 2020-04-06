---
layout: post
title: Till functions
date: 2013-09-19 20:56:55.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- F#
- folds
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1557131303;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4365;}i:1;a:1:{s:2:"id";i:4131;}i:2;a:1:{s:2:"id";i:4077;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/09/19/functions/"
---
Just wanted to share a couple little functions that I was playing with since it made my code terse and readable. At first I needed a way to fold a function until a predicate. This way I could stop and didn't have to continue through the whole list. Then I needed to be able to do the same kind of thing but choosing all elements up until a predicate.

## Folding

First, folding. I wanted to be able to get all the characters up until white space. For example:

[fsharp]  
let (++) a b = a.ToString() + b.ToString()

let upToSpaces str = foldTill Char.IsWhiteSpace (++) "" str  
[/fsharp]

Which led me to write the following fold function. Granted it's not lazy evaluated, but for me that was OK.

[fsharp]  
let foldTill check predicate seed list=  
 let rec foldTill' acc = function  
 | [] -\> acc  
 | (h::t) -\> match check h with  
 | false -\> foldTill' (predicate acc h) t  
 | true -\> acc  
 foldTill' seed list  
[/fsharp]

Running this gives

[fsharp]  
\> upToSpaces "abcdef gh";;  
val it : string = "abcdef"  
[/fsharp]

Here's a more general way of doing it for sequences. Granted it has mutable state, but its hidden in the function and never leaks. This is very similar to how fold is implemented in [F# core](https://github.com/fsharp/fsharp/blob/master/src/fsharp/FSharp.Core/seq.fs#L1042), I just added the extra check before it calls into the fold predicate

[fsharp]  
let foldTill check predicate seed (source:seq\<'a\>) =  
 let finished = ref false  
 use e = source.GetEnumerator()  
 let mutable state = seed  
 while e.MoveNext() && not !finished do  
 match check e.Current with  
 | false -\> state \<- predicate state e.Current  
 | true -\> finished := true  
 state  
[/fsharp]

Anyways, fun!

