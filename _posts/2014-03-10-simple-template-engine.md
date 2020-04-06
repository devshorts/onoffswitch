---
layout: post
title: A simple templating engine
date: 2014-03-10 08:00:07.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- F#
- fparsec
- java
- parsing
- tempating
- velocity
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554621603;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3710;}i:1;a:1:{s:2:"id";i:4306;}i:2;a:1:{s:2:"id";i:4737;}}}}

permalink: "/2014/03/10/simple-template-engine/"
---
I wanted to talk about templating, since templating is a common thing you run into. Often times you want to cleanly do a string replace on a bunch of text, and sometimes even need minimal language processing to do what you want. For example, Java has a templating engine called Velocity, but lots of languages have libraries that do this kind of work. I thought it'd be fun to create a small templating engine from scratch with F# as an after work exercise.

The goal is to give the templating processor a set of lookup bags that can be resolved by variables. For example, if I use a variable `$devshorts.isgreat` that should correspond to a bag that is keyed first off of `devshorts` which returns a new bag, and then a new bag that has a key `isgreat` which should return a value.

## Getting the AST

First, lets parse the language and get an abstract syntax tree. Anything that is prefixed with dollar sign is a language construct, anything not is a literal. As with most parsing tasks, I jump straight to fparsec.

[fsharp]  
namespace FPropEngine

module Parser =

open FParsec

type Ast =  
 | Bag of string list  
 | Literals of string  
 | ForLoop of string \* Ast \* Ast list

let tokenPrefix = '$'

let tagStart = pstring (string tokenPrefix)

let token n = tagStart \>\>. pstring n |\>\> ignore

let tagDelim = eof \<|\> spaces1

let endTag = token "end"

let forTag = token "for"

let languageSpecific = [attempt endTag; forTag] |\> List.map (fun i -\> i .\>\> tagDelim)

let anyReservedToken = attempt (languageSpecific |\> List.reduce (\<|\>))

let tokenable = many1Chars (satisfy isDigit \<|\> satisfy isLetter)

let element = attempt (tokenable .\>\> pstring ".") \<|\> tokenable

let nonTokens = many1Chars (satisfy (isNoneOf [tokenPrefix])) |\>\> Literals

let bag = tagStart \>\>. many1 element |\>\> Bag

let innerElement = notFollowedBy anyReservedToken \>\>. (nonTokens \<|\> bag)

let tagFwd, tagImpl = createParserForwardedToRef()

let forLoop = parse {  
 do! spaces  
 do! forTag  
 do! spaces  
 do! skipAnyOf "$"  
 let! alias = tokenable  
 do! spaces  
 let! \_ = pstring "in"  
 do! spaces  
 let! elements = bag  
 do! spaces  
 let! body = many tagFwd  
 do! spaces  
 do! endTag  
 do! spaces  
 return ForLoop (alias, elements, body)  
 }

tagImpl := attempt forLoop \<|\> innerElement

let get str =  
 match run (many tagFwd) str with  
 | Success(r, \_, \_) -\> r  
 | Failure(r,\_,\_) -\> failwith "nothing"  
[/fsharp]

I've exposed only one language construct (a for loop), and anything else is just a basic string replace bag (which will already be deconstructed into its individual components, i.e. `$foo.bar` will be `["foo";"bar"]`).

## Contexts

The next thing we need is a way to store a context, and to resolve a requested path from the context. Since I want to be able to add key value pairs to the context but have the values be different (sometimes they should be a string, other times they should be other context bags), we need to be able to handle that.

For example, lets say I make a context called "anton". In this context I want to have key "isGreat" that resolves to "kropp". That would end up being a leaf node in this context path. But how do I represent a path like "anton.shmanton.isGreat". The key "shmanton" should resolve to a new context under the current context of "anton". Also, in order to leverage for loops, we need some keys to resolve to multiple values. So now we have 3 types of results: a string, a string list, or another context. Given that, lets create a context class that can handle creating these contexts, as well as resolving a context path.

[fsharp]  
module Formatter =  
 open Parser  
 open System.Collections.Generic

type Context () =  
 let ctxs = new Dictionary\<string, ContextType\>()  
 let runtime = new Dictionary\<string, string\>()

member x.add (key, values) = ctxs.[key] \<- List values  
 member x.add (key, value) = ctxs.[key] \<- Value value  
 member x.add (key, ctx) = ctxs.[key] \<- More ctx

member x.runtimeAdd (key, value) = runtime.[key] \<- value  
 member x.runtimeRemove key = runtime.Remove key |\> ignore

member x.add (dict:Dictionary\<string, string\>) =  
 for keys in dict do  
 ctxs.[keys.Key] \<- Value keys.Value

member x.resolve list =  
 match list with  
 | [] -\> None  
 | h::t -\>  
 if runtime.ContainsKey h then  
 Some [runtime.[h]]  
 else if ctxs.ContainsKey h then  
 ctxs.[h].resolve t  
 else  
 None

and ContextType =  
 | Value of string  
 | List of string list  
 | More of Context  
 member x.resolve list =  
 match x with  
 | Value str -\> Some [str]  
 | List strs -\> Some strs  
 | More ctx -\> ctx.resolve list  
[/fsharp]

One thing that is tricky here: `ctxs.[h].resolve t` doesn't call the same `resolve` function on the Context class. It actually calls the resolve function on the ContextType. This way each type can resolve itself. If you call resolve on a string, it'll return itself (as a list). If you resolve on a list, it'll return the list. But, if you call resolve on a context, it'll proxy that request back to the Context class.

You may also be wondering what "runTimeAdd" and "runtimeRemove" are. Those will make sense when we actually create the language interpreter. It may be a little overkill to call this a "language" but it kind of is!

## Applying the context to the AST

Now we need to interpret the syntax tree and apply the context bag to any context related tokens. If anybody read my previous posts about my language I wrote, this should all sound pretty similar (cause it is!)

[fsharp]  
module Runner =  
 open Formatter  
 open Parser

let rec private eval (ctx : Context) = function  
 | Bag list -\>  
 match ctx.resolve list with  
 | Some item -\> item  
 | None -\> [List.fold (fun acc i -\> acc + "." + i) "$" list]  
 | Literals l -\> [l]  
 | ForLoop (alias, bag, contents) -\>  
 [for value in (eval ctx bag) do  
 ctx.runtimeAdd (alias, value)  
 for elem in contents do  
 yield! eval ctx elem  
 ctx.runtimeRemove alias]

let run ctx text =  
 Parser.get text  
 |\> List.map (eval ctx)  
 |\> List.reduce List.append  
 |\> List.reduce (+)  
[/fsharp]

What we have here is an eval function that acts as the main interpreter dispatch loop. It's asked to evaluate the current token its given based on its current context.

If we have a string literal, we just return it (as a list, since I am creating a list of evaluated results).

If there is a bag (like `$anton.isgreat`) then try and resolve the bag path from the context.

If there is a for loop we want to evaluate the result of the for predicate and bind its value to the alias. Then for each element we want to evaluate the contents of the for loop. This is where we need to create a runtime storage of the alias, so we can do later lookups in the context. You can see that each for loop adds its alias to the context and then removes it from the context afterwards. This would mimic a regular language where inner loops can access outer declared variables, but not vice versa.

## Trying it out

Let's give our templating engine a whirl:

[fsharp]

let artists = new Context()  
let root = new Context()

artists.add("nirvana", ["come as you are";"smells like teen spirit"]);  
root.add("artists", artists );

let templateText = "$for $song in $artists.nirvana  
 The current song is $song!  
 $for $secondTime in $artists.nirvana  
 Oh lets just loop again for fun. First value: $song, second: $secondTime  
 $end  
 $end"

[/fsharp]

And the result is

[fsharp]  
\> Runner.run root templateText;;  
val it : string =  
 "The current song is come as you are!  
 Oh lets just loop again for fun. First value: come as you are, second: come as you are  
 Oh lets just loop again for fun. First value: come as you are, second: smells like teen spirit  
 The current song is smells like teen spirit!  
 Oh lets just loop again for fun. First value: smells like teen spirit, second: come as you are  
 Oh lets just loop again for fun. First value: smells like teen spirit, second: smells like teen spirit  
 "  
[/fsharp]

Not too bad!

Full source available at my [github](https://github.com/devshorts/Playground/tree/master/FPropBag)

