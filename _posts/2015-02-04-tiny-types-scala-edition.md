---
layout: post
title: Tiny types scala edition
date: 2015-02-04 21:30:15.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561511191;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4905;}i:2;a:1:{s:2:"id";i:4862;}}}}

permalink: "/2015/02/04/tiny-types-scala-edition/"
---
Previously I wrote about generating value type wrappers on top of C# primitives for better handling of domain level knowledge. This time I decided to try it out in scala as I'm jumping into the JVM world.

With scala we don't have the value type capability that c# has, but we can sort of get there with implicits and case classes.

The simple gist is to generate stuff like

```scala
  
package com.devshorts.data

case class foo(data : String)  
case class bar(data : String)  
case class bizBaz(data : Int)  
case class Data(data : java.util.UUID)  

```

And the implicit conversions

```scala
  
package com.devshorts.data

object Conversions{  
 implicit def convertfoo(i : foo) : String = i.data  
 implicit def convertbar(i : bar) : String = i.data  
 implicit def convertbizBaz(i : bizBaz) : Int = i.data  
 implicit def convertData(i : Data) : java.util.UUID = i.data  
}  

```

Now you get a similar feel of primitive wrapping with function level unboxing and you can pass your primitive case class wrappers to more generic functions.

For this case I wrote a simple console generator and played around with zsh auto completion for it too. Full source located at my [github](https://github.com/devshorts/scala-tiny-types)

