---
layout: post
title: Capturing union values with fparsec
date: 2013-05-02 20:02:06.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- combinators
- fsharp
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _wp_old_slug: fparsec
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558731860;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4131;}i:1;a:1:{s:2:"id";i:4068;}i:2;a:1:{s:2:"id";i:4077;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/02/capturing-union-values-fparsec/"
---
I just started playing with [fparsec](http://www.quanttec.com/fparsec/) which is a [parser combinatorics](http://en.wikipedia.org/wiki/Parser_combinator) library that lets you create chainable parsers to parse DSL's. After having built my own parser, lexer, and interpreter, playing with other libraries is really fun, I like seeing how others have done it. Unlike my mutable parser written in C#, with FParsec the idea is that it will encapsulate the underlying stream state and result into a parser object. Since F# is mostly immutable, this is how the underlying modified stream state gets captured and passed as a new stream to the next parser. I actually like this kind of workflow since you don't need to create a grammar which is parsed and creates code for you (which is what ANTLR does). There's something very appealing to have it be dynamic.

As a quick example, I was following the [tutorial](http://www.quanttec.com/fparsec/tutorial.html) on the fparsec site and wanted to understand how to capture a value to store in a discriminated union. For example, if I have a type

[fsharp]  
type Token =  
 | Literal of string  
[/fsharp]

How do I get a `Literal("foo")` created?

All of the examples I saw never looked to instantiate that. After a bit of poking around I noticed that they were using the `|>>` syntax which is a function that is passed the result value of the capture. So when you do

[fsharp]  
pstring "foo" |\>\> Literal  
[/fsharp]

You've invoked the constructor of the discriminated union similar to this:

[fsharp]  
let literal = "foo" |\> Literal  
[/fsharp]

Which is equivalent to

[fsharp]  
let literal = Literal("foo")  
[/fsharp]

This is because most of the fparsec functions and overloads give you back a Parser type

[fsharp]  
type Parser\<'TResult, 'TUserState\> = CharStream\<'TUserState\> -\> Reply\<'TResult\>  
[/fsharp]

Which is just an alias for a function that takes a utf16 character stream that holds onto a user state and returns a reply that holds the value you wanted. If you look at [charstream](http://www.quanttec.com/fparsec/reference/charstream.html#CharStream) it looks similar to my simple [tokenizer](https://github.com/devshorts/LanguageCreator/blob/master/Lang/Lexers/TokenizableStreamBase.cs). The functions `|>>`, `>>=`, and `>>%` are all overloads that help you chain parsers and get your result back. If you are curious you can trace through their types [here](http://www.quanttec.com/fparsec/reference/primitives.html#members.:62::62::61:).

Now, if you don't need to capture the result value and want to just create an instance of an empty union type then you can use the `>>%` syntax which will let you return a result:

[fsharp]  
let Token =  
 | Null  
let nullTest = pstring "null" \>\>% Null  
[/fsharp]

There are a bunch of overloaded methods and custom operators with fparsec. For example

[fsharp]  
let Token =  
 | Null  
let nullTest = stringReturn "null" Null  
[/fsharp]

Is equivalent to the `>>%` example.

It's a little overwhelming trying to figure out how all the combinators are pieced together, but that's part of the fun of learning something new.

