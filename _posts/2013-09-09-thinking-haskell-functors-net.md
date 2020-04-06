---
layout: post
title: Thinking about haskell functors in .net
date: 2013-09-09 20:06:52.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- functors
- haskell
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558842116;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4348;}i:1;a:1:{s:2:"id";i:4725;}i:2;a:1:{s:2:"id";i:4262;}}}}

permalink: "/2013/09/09/thinking-haskell-functors-net/"
---
I've been teaching myself haskell lately and came across an interesting language feature called functors. Functors are a way of describing a transformation when you have a boxed container. They have a generic signature of

[fsharp]  
('a -\> 'b) -\> f 'a -\> f 'b  
[/fsharp]

Where `f` isn't a "function", it's a type that contains the type of `'a`.

The idea is you can write custom map functions for types that act as generic containers. Generic containers are things like lists, an option type, or other things that _hold_ something. By itself a `list` is nothing, it has to be a list OF something. Not to get sidetracked too much, but these kinds of boxes are called Monads.

Anyways, let's do this in C# by assuming that we have a box type that holds something.

[csharp]

public class Box\<T\>  
{  
 public T Data { get; set; }  
}

var boxes = new List\<Box\<string\>\>();

IEnumerable\<string\> boxNames = boxes.Select(box =\> box.Data);

[/csharp]

We have a type `Box` and a list of `boxes`. Then we `Select` (or map) a box's inner data into another list. We could extract the projection into a separate function too:

[csharp]  
public string BoxString(Box\<string\> p)  
{  
 return p.Data;  
}  
[/csharp]

The type signature of this function is

[csharp]  
Box-\> string  
[/csharp]

But wouldn't it be nice to be able to do work on a boxes data without having to explicity project it out? Like, maybe define a way so that if you pass in a box, and a function that works on a string, it'll automatically unbox the data and apply the function to its data.

For example something like this (but this won't compile obviously)

[csharp]  
public String AddExclamation(String input){  
 return input + "!";  
}

IEnumerable\<Box\<string\>\> boxes = new List\<Box\<string\>\>();

IEnumerable\<string\> boxStringsExclamation = boxes.Select(AddExclamation);  
[/csharp]

In C# we have to add the projection step (which in this case is overloaded):

[csharp]  
public String AddExclamation(Box\<String\> p){  
 return AddExclamation(p.Data);  
}  
[/csharp]

In F# you have to do basically the same thing:

[fsharp]  
type Box\<'T\> = { Data: 'T }

let boxes = List.init 10 (fun i -\> { Data= i.ToString() })

let boxStrings = List.map (fun i -\> i.Data) boxes  
[/fsharp]

But in Haskell, you can define this projection as part of the type by saying it is an instance of the `Functor` type class. When you make a generic type an instance of the functor type class you can define how maps work on the insides of that class.

[fsharp]  
data Box a = Data a deriving (Show)

instance Functor Box where  
 fmap f (Data inside) = Data(f inside)

main =  
 print $ fmap (++"... your name!") (Data "my name")  
[/fsharp]

This outputs

[code]  
Data "my name... your name!"  
[/code]

Here I have a box that contains a value, and it has a value. Then I can define how a box behaves when someone maps over it. As long as the type of the box contents matches the type of the projection, the call to `fmap` works.

