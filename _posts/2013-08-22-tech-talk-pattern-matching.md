---
layout: post
title: 'Tech talk: Pattern matching'
date: 2013-08-22 16:00:40.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- F#
- pattern matching
- tech talk
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560233663;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3565;}i:1;a:1:{s:2:"id";i:4961;}i:2;a:1:{s:2:"id";i:4881;}}}}

permalink: "/2013/08/22/tech-talk-pattern-matching/"
---
Today's tech talk was about functional pattern matching. This was a really fun one since I've been sort of "evangelizing" functional programming at work, and it was a blast seeing everyone ask poignant and intersting questions regarding pattern matching.

What spurred the conversation today was a question my boss asked me which was "how is pattern matching actually compiled?" which led me to find [this blog post](http://www.codeproject.com/Articles/520869/A-Simple-Overview-on-How-Pattern-Match-Compiles) describing different ways the f# compiler compiles pattern matching. The short of it is that the compiler generates a souped up switch statement where it checks each pattern in order. Sometimes it does a good job, sometimes it doesn't, but that's OK.

In the process of researching for the tech talk I came across a great paper entitled [Pattern Matching in Scala](http://wiki.ifs.hsr.ch/SemProgAnTr/files/PatternMatchingInScala.pdf) which discussed, obviously, pattern matching in Scala, but also talked about F#, Haskell, and Erlang pattern matching. The interesting thing to me here is how Scala got around comparing classes instead of just algebraic data types. Scala makes you implement classes as specific `case` classes when you want to be able to match on them, and also you have to implement `apply` and `unapply` methods which effectively "box" and "unbox" your pattern.

I don't have much experience with Scala (I skimmed a Scala book and wrote a hello world, but that's it), but I am familiar with how F# handled this scenario which is via Active Patterns. I like this since you can mix and match active patterns to provide your own custom way to "compare" items.

An example I used in our talk today was

[fsharp]  
let (|Pattern1|\_|) i =  
 if i = 0 then Some(Pattern1) else None

let (|Pattern2|\_|) i =  
 if i.ToString() = "yo mamma!" then Some(Pattern2) else None

let activePatternTest () =  
 let x = 0  
 match x with  
 | Pattern1 -\> printf "pattern1"  
 | Pattern2 -\> printf "pattern2"  
 | \_ -\> printf "something else"  
[/fsharp]

Which really drives the point home that you can do custom work in your pattern match and hide it away from the user. Another, more real world, example is how I matched on regular expressions in my [parsec clone](https://github.com/devshorts/ParsecClone) project

[fsharp]  
let (|RegexStr|\_|) (pattern:string) (input:IStreamP\<string, string\>) =  
 if String.IsNullOrEmpty input.state then None  
 else  
 let m = Regex.Match(input.state, "^(" + pattern + ")", RegexOptions.Singleline)  
 if m.Success then  
 Some ([for g in m.Groups -\> g.Value]  
 |\> List.filter (String.IsNullOrEmpty \>\> not)  
 |\> List.head)  
 else  
 None  
[/fsharp]

Which can be used to hide away regular expression pattern matching. The usage of this would now be:

[fsharp]  
member x.regexMatch (input:IStreamP\<string, string\>) target =  
 if String.IsNullOrEmpty input.state then None  
 else  
 match input with  
 | RegexStr target result -\> Some(result.Length)  
 | \_ -\> None  
[/fsharp]

Nice and clean, just the way I like it.

Anyways, pattern matching is a really powerful construct and it's a shame that it's not available in many OO languages.

