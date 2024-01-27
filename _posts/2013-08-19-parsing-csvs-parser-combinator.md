---
layout: post
title: Parse whatever with your own parser combinator
date: 2013-08-19 08:00:32.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- ast
- combinators
- F#
- fparsec
- parsing
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558681316;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4068;}i:1;a:1:{s:2:"id";i:3723;}i:2;a:1:{s:2:"id";i:4213;}}}}

permalink: "/2013/08/19/parsing-csvs-parser-combinator/"
---
In a few recent posts I talked about playing with fparsec to parse data into usable syntax trees. But, even after all the time spent fiddling with it, I really didn't fully understand how combinators actually worked. With that in mind, I decided to build a version of fparsec from scratch. What better way to understand something than to build it yourself? I had one personal stipulation, and that was to not look at the fparsec source. To be fair, I cheated with one function (the very first one) so I kind of cheated a lot, but I didn't peek at anything else, promise.

## Combinators

The principle behind combinators is that they are a way to take two functions and combine them into another function. Functional programming is chock full of this pattern. In general, you can combine any function to get any other function, but what makes a combinator powerful is when you combine a function, with another function, and get the same signature as the first function. Now you can recursively combine functions together.

For example, lets say I have a function defined like this:

```fsharp
  
let parser = state -\> result \* state  

```

So it takes some input state, and gives you some sort of result along with a new state.

To make this useful though, I need to get the result and do something else with it. So, lets say I have another function that takes a result, and gives you a function that takes a new state and returns a new result.

```fsharp
  
let applier = result -\> (state -\> result \* state)  

```

Is it possible to combine these two somehow? Sure:

```fsharp
  
let combiner parser applier =  
 fun state -\>  
 match parser state with  
 | (Some(result), newState) -\>  
 let nextParser = applier result  
 nextParser newState  
 | (None, newState) -\> (None, newState)  

```

What's the function signature of this combiner function?

```fsharp
  
(state -\> result \* state) -\> (result -\> (state -\> result \* state)) -\> (state -\> result \* state)  

```

That's kind of a mouthful, so lets add a type alias:

```fsharp
  
type Parser = state -\> result \* state  

```

Now what is the type signature?

```fsharp
  
Parser -\> (result -\> Parser) -\> Parser  

```

That's a lot better.

We've just defined a way to take some function that takes a state and returns a result, combine it with something that takes a result and returns a new function, and it gives you a NEW parser. So you've combined the two things together to create the same kind of thing as the first thing!

What's neat about this is you can use this basic `combiner` function to build up parsers that do small work.

## Define a simple parser

Building a parser function is easy, it can be anything. Remembering the signature:

```fsharp
  
state -\> result \* state  

```

Here is something that parsers a single character from a string:

```fsharp
  
let oneCharParser =  
 fun (state:string) -\>  
 let firstChar = state.Chars(0)  
 let remainingState = state.Substring(0, state.Length - 1)  
 (Some(firstChar), remainingState)  

```

You can create more complex parsers too, maybe one that takes a regular expression or matches on a specific string. Anything you want.

## Building on the combiner

Now that there is a combiner, and a way to define a parser, lets build on that. First lets alias the `combiner` function I first wrote out to make it a little easier to use:

```fsharp
  
let (\>\>=) current next = combiner current next  

```

This operator, mimics the syntax from fparsec (and haskells parsec). Next, lets make a combinator function that takes two parsers, and returns the result of the second parser (ignoring the result of the first):

```fsharp
  
let (\>\>.) parser1 parser2 =  
 parser1 \>\>= fun firstResult -\>  
 parser2 \>\>= fun secondResult -\>  
 secondResult  

```

But wait, that won't really work. Remember that the combiners second argument wants a function that takes a result and returns a parser (which is a function that takes a state and returns a result) [i.e. of the signature `result -> (state -> result * state)`.

If you look closely, we aren't returning a parser at the end of this, we are just returning a value (i.e. just `result`). We need some sort of way to return a value as a parser. Hmm, fparsec has this and its called `preturn`. Let's add this too:

```fsharp
  
let preturn value = fun state -\> (Some(value), state)  

```

Not so bad. Now lets tie it in:

```fsharp
  
let (\>\>.) parser1 parser2 =  
 parser1 \>\>= fun firstResult -\>  
 parser2 \>\>= fun secondResult -\>  
 preturn secondResult  

```

Awesome, now the magic that is the F# type inference system is happy!

But, we can do so much more. If all we need to do is create custom operators that leverage the combiner function we can create functions that:

- Takes two parsers and returns the first (`.>>`)
- Takes two parsers and returns the second (`>>.`)
- Takes two parsers and returns a tuple of the result (`.>>.`)
- Takes a parser and preturns its value into a parameterized discriminated union type (`|>>`)
- Takes a parser and preturns its value into a non parameterized discriminated union type (`|>>%`)

## State

In FParsec and other combinators, the state is a character stream. But when building out my own parsec clone I saw no reason that the state had to be tied to a specific type. Parser states share a few things in common:

- Consume and return a consumed value
- Backtrack to a position
- Test if the state contains a predicate
- Know if they are empty

When building my parser I kept this in mind and made the combinator library work on a general state interface that I called `IStreamP`

```fsharp
  
type IStreamP\<'StateType, 'ConsumeType\> =  
 abstract member state : 'StateType  
 abstract member consume : int -\> 'ConsumeType option \* IStreamP\<'StateType, 'ConsumeType\>  
 abstract member backtrack : unit -\> unit  
 abstract member hasMore : unit -\> bool  

```

Using this interface I was able to implement a string parser state, as well as a binary parser state. Both states can be reused with all the combinator functions which is part of what makes parser combinators so robust. You get to mix language functionality with the grammar you are parsing.

## An example

Now that I have all the basic building blocks, lets try parsing a CSV. The bulk of the work is being able to parse a string. But to parse a string, we have to see if the state matches something. So lets start with that. The combinator I wrote has a generic match function that you inject a predicate to:

```fsharp
  
let matcher eval target =  
 fun currentState -\>  
 match eval currentState target with  
 | Some(amount) -\> currentState.consume amount  
 | None -\> (None, currentState)  

```

The signature of the eval function is:

```fsharp
  
state -\> 'a -\> int  

```

So an evaluator function takes the current state, and some sort of target (maybe you are trying to match on a specific string) and if the predicate returns some integer amount, the state consumes the amount the predicate told it to take. A simple way of doing this is to see if the beginning of the string matches what you want to take:

```fsharp
  
type ParseState = State\<string, string\>

let private getStringStream (state:ParseState) = (state :?\> StringStreamP)

let private startsWith (input:ParseState) target = (input |\> getStringStream).startsWith input target

let matchStr str = matcher startsWith str  

```

Basically its just getting a starts with expression match function from the state class. The idea here is that each state class can contain its own predicates to match on, so you don't have to mix stuff between a binary state parser and a string state parser.

And just to show what the startsWith function looks like:

```fsharp
  
member x.startsWith (inputStream:IStreamP\<string, string\>) target =  
 if String.IsNullOrEmpty inputStream.state then None  
 else if inputStream.state.StartsWith target then  
 Some target.Length  
 else None


```

Building on this we can create matches that match using regex, or do other work. Taking kind of a leap of faith here, let me show the finished CSV parser (full source is on my [github](https://github.com/devshorts/ParsecClone))

```fsharp
  
let delimType = ","

let(|DelimMatch|EscapedType|Other|) i =  
 if i = "\\" || i ="\"" then EscapedType  
 else if i = delimType then DelimMatch  
 else Other

let delim\<'a\> = matchStr delimType

let quote = matchStr "\""

let validNormalChars = function  
 | EscapedType  
 | DelimMatch -\> false  
 | rest -\> not (isNewLine rest)

let inQuotesChars = function  
 | "\"" -\> false  
 | \_ -\> true

let unescape = function  
 | "n" -\> "\n"  
 | "r" -\> "\r"  
 | "t" -\> "\t"  
 | c -\> c

let quoteStrings = (many (satisfy (inQuotesChars) any)) \>\>= foldChars

let escapedChar\<'a\> = matchStr "\\" \>\>. (anyOf matchStr [delimType; "\"";"n";"r";"t"] |\>\> unescape)

let normal\<'a\> = satisfy validNormalChars any

let normalAndEscaped = many (normal \<|\> escapedChar) \>\>= foldChars

let literal\<'a\> = between quote quoteStrings quote

let csvElement = ws \>\>. (literal \<|\> normalAndEscaped)

let listItem\<'a\> = delim \>\>. opt csvElement

let elements\<'a\> = csvElement .\<?\>\>. many listItem

let lines\<'a\> = many (elements |\> sepBy \<| newline) .\>\> eof  

```

It should look very similiar to fparsec, but slightly different. For example, the ```
.\<?\>\>.
``` operator takes an item parser, and an item list parser, and optionally applies both the item and the list. If the list returns any results it preturns the first item with the item list, otherwise just returns the first item.

## Another example

Just to demonstrate the power of decoupling the combinator logic from the state/stream logic, lets use the same combinator functions on a binary stream:

If we implement a new binary state stream, it might look like this:

```fsharp
  
type BinStream (state:Stream) =  
 let startPos = state.Position

interface IStreamP\<Stream, byte[]\> with  
 member x.state = state

member x.consume (count) =  
 let mutable bytes = Array.init count (fun i -\> byte(0))  
 state.Read(bytes, 0, count) |\> ignore

(Some(bytes), new BinStream(state) :\> IStreamP\<Stream, byte[]\> )

member x.backtrack () = state.Seek(startPos, SeekOrigin.Begin) |\> ignore

member x.hasMore () = state.Position \<\> state.Length

member x.streamCanBeConsumed (state:IStreamP\<Stream, byte[]\> ) count =  
 if (int)state.state.Position + (int)count \<= (int)state.state.Length then  
 Some(count)  
 else  
 None  

```

And we can define a whole bunch of basic parsers to work with this stream:

```fsharp
  
module BinParser =

let private byteToInt (b:byte) = System.Convert.ToInt32(b)  
 let private toInt16 v = System.BitConverter.ToInt16(v, 0)  
 let private toInt32 v = System.BitConverter.ToInt32(v, 0)  
 let private toInt64 v = System.BitConverter.ToInt64(v, 0)

type ParseState = State\<Stream, byte[]\>

let private getBinStream (state:ParseState) = (state :?\> BinStream)

let private streamCanBeConsumed (state:ParseState) count = (state |\> getBinStream).streamCanBeConsumed state count

let private binMatch (num:int) = matcher streamCanBeConsumed num

let byteN\<'a\> = binMatch

let byte1\<'a\> = byteN 1 \>\>= fun b1 -\> preturn b1.[0]

let byte2\<'a\> = byteN 2

let byte3\<'a\> = byteN 3

let byte4\<'a\> = byteN 4

let int16\<'a\> = byte2 |\>\> toInt16

let int32\<'a\> = byte4 |\>\> toInt32

let int64\<'a\> = byteN 8 |\>\> toInt64

let intB\<'a\> = byte1 |\>\> byteToInt  

```

And here is a unit test to show how it might be used:

```fsharp
  
[\<Test\>]  
let ``test reading two sets of 4 bytes``() =  
 let bytes = [|0;1;2;3;4;5;6;7;8|] |\> Array.map byte

let stream = new MemoryStream(bytes)

let binaryStream = new BinStream(stream)

let parser = manyN 2 byte4

let result = test binaryStream parser

result |\> should equal [[|0;1;2;3|];[|4;5;6;7|]]  

```

## Conclusion

What I like about combinators is that you build on the smallest blocks. And, unlike parser generators, you can mix language constructs with your grammar. Unfortunately debugging combinators is extremely difficult, since each combinator is a function that is composed of other functions. When you build a complex grammar up from those blocks, its easy to get lost in which function you are in and where you came from. The up side is that you can easily test against each building block independently.

