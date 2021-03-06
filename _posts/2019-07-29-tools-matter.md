---
layout: post
title: Tools Matter
date: 2019-07-29 22:11:40.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags: []
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none

permalink: "/2019/07/29/tools-matter/"
---
<!-- wp:paragraph -->

I feel like I've written about this before, but it continues to come up in my career. Tools matter. Without them you can't be an effective engineer. When I bring this up sometimes some intrepid person will give me the "_it's a poor craftsman_" schpiel, but have you ever met a craftsperson who doesn't care deeply about their tools? Ask any woodworker and they will tell you the countless hours fine tuning, honing, sharpening, and improving their tools. Without them, the job will take longer, be done poorer, and cost more.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

In my mind the biggest most important tool of all for an engineer is their IDE. Lots of people have opinions on what flavor editor/IDE they want but given it's nearly 2020, I think there are a few critical tooling features that **must work** otherwise they are a non starter.

<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->

1. **Go to definition**. It seems crazy to me that I have to spell this out, but code is data. Any editor should be able to _semantically_ tell you the source of a method/class/function/variable. Grep is not that. Languages are context sensitive, and thats why you need a context sensitive system to help navigate it
2. **Find references**. Same as above, just like you can find the definition, you need to find who is using it. Again its context sensitive, I don't want to someone to convince me that vim ctags or grep are good enough. They. Are. Not.
3. **Semantic Autocomplete**. CTAGS are not semantic autocomplete. CTAGS are the type of completion where an editor indexes the current file(s) and autocompletes based on what's in the file. I see many developers using this and it pains me to no end. As an engineer my time isn't worth reading scaladoc or ruby docs to find what arguments are to `map`. I need to hit dot and have the IDE tell me. And if I'm wrong, the IDE should squiggle it and tell me I'm wrong so I can fix it fast and move on. I'm not getting paid to be a compiler. That's a compilers job.
4. **Semantic Rename and Refactor**. Again, semantics are key. Sed is not semantic. You need to be able to do repo wide renames, refactors, moves/etc, with confidence that the AST is still properly intact and semantically rigorous. 
5. **Integrated debugging and test running**. Context switching to the CLI to run tests, and in particular a single test, is insane. While many would argue that test runners and debuggers can be done out of band, and they're not _wrong_, but they are missing the fact that if you are doing this task hundreds of times a day (easily) then each moment spent typing or context switching adds up. In an incident any of those things that slow you down matter and have a monetary impact.

<!-- /wp:list -->

<!-- wp:paragraph -->

What do I mean by "semantic"? Using common CLI tools like sed, grep, is it possible to disambiguate a search for "Foo" in the context of A vs the context of B?:

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
class A { def Foo() String } class B { def Foo() String }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Could you refactor this:

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
class C(a A, b B){ def Foo() String { a.Foo() b.Foo() } }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

To be

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
class C(a A, b B){ def Foo() String { a.Bar() b.Foo() } }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

You can't. And yet these situations arise constantly in day to day engineering. Good code is refactored code, and codebases that don't get constantly improved upon rot and fall apart.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

I feel like these requirements are so basic and primitive that I'm constantly shocked when I encounter organizations that run without them!

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

My thoughts are that anytime systems are being built we need to make sure that the tools continue to work with them. Tooling is important not because I'm too stupid to be a l337 hacker on the CLI, but because I'm smart enough to know that I don't need to waste my time on that.

<!-- /wp:paragraph -->

