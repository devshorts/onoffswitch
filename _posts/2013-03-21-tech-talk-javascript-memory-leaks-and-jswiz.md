---
layout: post
title: 'Tech talk: Javascript Memory Leaks and JSWhiz'
date: 2013-03-21 18:12:30.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- garbage collection
- JavaScript
meta:
  _edit_last: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558691246;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2635;}i:1;a:1:{s:2:"id";i:4394;}i:2;a:1:{s:2:"id";i:3710;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/21/tech-talk-javascript-memory-leaks-and-jswiz/"
---
Todays tech talk revolved around the recently published [JSWhiz whitepaper](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/en/us/pubs/archive/40738.pdf) from google. The paper discusses common javascript memory leak patterns. It also goes over how those leaks can be created and how google automated detection of them using [Closure](https://developers.google.com/closure/) type annotations.

## JSWhiz

First Google identified common javascript memory leak patterns, though some of these are described in terms of Closure syntax

- Create without dispose EventHandler - "_Closure objects of type EventHandler provide a simple mechanism to group all events and listeners associated with an object together. This allows for easy removal of listeners when the events being listened to can no longer ﬁre. Unfortunately this is also one of the main sources of leaks_"
- Setting member to null without removing event listeners. "_In JavaScript setting a variable to null is the idiomatic way of providing a hint to the garbage collector that the memory allocated by an object can be reclaimed. But the EventHandler and event listeners attached to the object prevents the garbage collector from disposing of the object and reclaiming memory._" 
- Undisposed member object has EventHandler as member
- Object graveyard (holding onto objects in a persistent structure like a list/dictionary so objects can't ever get collected)
- Overwriting EventHandler ﬁeld in derived class. - "_Closure implements inheritance using a prototype-based approach [8], which can break the expectations of programmers coming from a more standard OOP language background (such as C++ or Java). For EventHandlers this can cause memory leaks as the overwritten EventHandler cannot be freed [8]._"
- Unmatched listen/unlisten calls. - "_The semantics of the listen and unlisten calls require all parameters to be the same. When the parameters do not match, the event listener is not removed and a reference to objects being listened remains, inhibiting the garbage disposal._"
- Local EventHandler - "_A local EventHandler instance that does not escape scope, e.g., a locally deﬁned variable that is not assigned to a ﬁeld member, added to an array, or captured in a closure, can not be disposed of later._"

## How

Google's engineers leveraged the fact that they had an annotated AST of javascript (with the Closure compiler) to look these particular patterns and help identify potential leaks. It's not always as easy as it sounds though, as the paper mentions in one section

> Doing the analysis in a ﬂow-insensitive manner is equal to assuming that an eventful object is disposed of if there exists a disposal instruction in the AST tree. This assumption is further necessitated by 1) the dynamic nature of JavaScript, 2) most disposals resulting from runtime events (user actions, that may or may not occur) and 3) the computational demands associated with full program/inter-procedural analysis.

Other pitfalls of their current analysis include

- Not tracking objects with fully qualified names (such as application.window.toolbar)
- Objects that are never returned
- Objects not captured in closures
- Not checking of double disposal

Still, it's an impressive feat.

## Discussion

After running through the paper we started talking about different garbage collection issues with javascript. One of the big problems people have encountered is that since the DOM and javascript use reference counting you can easily get yourself into [circular references](http://www.ibm.com/developerworks/web/library/wa-memleak/) which would prevent collection. We talked a bit about generational collection since that's how the [V8 engine](https://developers.google.com/v8/design#garb_coll) that chrome and node.js use, and how that can solve circular references by doing path from root detection.

A couple other neat points came up, such as that the `delete` keyword [doesn't actually delete memory](http://buildnewgames.com/garbage-collector-friendly-code/). It only deletes properties of an object. Just to double check that we found ourselves at the [mozilla dev site](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Operators/delete) which had this hilariously appropriate quote on the page for the `delete` keyword

> The delete operator removes a property from an object. Unlike what common beliefs suggests, the delete operator has nothing to do with directly freeing memory (it only does indirectly via breaking references.

## Conclusion

In the end we decided that these memory leak patterns aren't strictly related to javascript. Lots of the same issues occur with actionscript and other languages (even .NET), where you have strong references in the form of event handlers or closures and things can never be garbage collected.

The ultimate way to avoid memory leaks is to be cognizant of who uses what, who holds on to what, and what needs to be removed when.

