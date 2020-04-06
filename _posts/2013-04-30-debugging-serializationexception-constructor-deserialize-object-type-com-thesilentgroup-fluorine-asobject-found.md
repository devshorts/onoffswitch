---
layout: post
title: 'Debugging Serialization Exception: The constructor to deserialize an object
  was not found.'
date: 2013-04-30 20:38:27.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- Debugging
- serialization
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560848118;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4456;}i:1;a:1:{s:2:"id";i:4919;}i:2;a:1:{s:2:"id";i:3779;}}}}

permalink: "/2013/04/30/debugging-serializationexception-constructor-deserialize-object-type-com-thesilentgroup-fluorine-asobject-found/"
---
Today I was debugging an exception that was occuring when remoting a data object between two .NET processes. I kept getting

[code]  
System.Runtime.Serialization.SerializationException: The constructor to deserialize an object of type 'com.TheSilentGroup.Fluorine.ASObject' was not found.  
[/code]

The issue I had was that there was a .NET object that looked like this

[csharp]  
public class ItemDto{  
 public object Item { get;set; }  
}  
[/csharp]

Which was used as a [covariant](http://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) store for any item (because everything is an `object`). This was needed because the code I was working in leveraged reflection to pull out certain fields at runtime depending on whatever this object type really was.

But, at some point the `ItemDto` object was sent to an ActionScript frontend. Later, the same object came back to .NET and the property `Item` was now a `ASObject` type due to [Fluorine](http://www.fluorinefx.com/)'s serialization process. Next, this object had to be serialized to another .NET process, and that's where the exception was happening.

It was obvious that there was an issue with the `ASObject` but I had assumed it was because the host process (that I was remoting to) didn't have a reference to Fluorine's dll to serialize the ASObject. But that wasn't it.

Turns out that `ASObject` inherits from `HashTable` which implements `ISerializable`.

[![HashTable ISerializable](http://onoffswitch.net/wp-content/uploads/2013/04/2013-04-30-16_31_02-JetBrains-dotPeek-1.0-EAP.-Build-1.0.0.png)](http://onoffswitch.net/wp-content/uploads/2013/04/2013-04-30-16_31_02-JetBrains-dotPeek-1.0-EAP.-Build-1.0.0.png)

From the [MSDN](http://msdn.microsoft.com/en-us/library/system.runtime.serialization.iserializable.aspx)

> If a class needs to control its serialization process, it can implement the ISerializable interface. [...] The ISerializable interface **implies a constructor** with the signature constructor (SerializationInfo information, StreamingContext context). At deserialization time, the current constructor is called only after the data in the SerializationInfo has been deserialized by the formatter. In general, this constructor should be protected if the class is not sealed.

So, because an interface can't enforce a constructor signature, you have to make sure to implement this constructor on any object (even derived objects) if they are going to be serialized across .NET boundaries and implement ISerializable.

The solution, for me, was to just not send this object to the UI and back. It didn't need to. But, you can overcome this issue by subclassing your data class, implementing the needed constructor, and calling the base class (in this case HashTable's) constructor for proper serialization.

