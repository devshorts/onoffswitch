---
layout: post
title: Bit packing Pacman
date: 2017-04-08 23:28:11.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- low-level
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561898908;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3735;}i:1;a:1:{s:2:"id";i:3477;}i:2;a:1:{s:2:"id";i:4011;}}}}
  _wpcom_is_markdown: '1'
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'
  _wp_old_slug: bit-packing

permalink: "/2017/04/08/bit-packing-pacman/"
---
Haven't posted in a while, since I've been heads down in building a lot of cool tooling at work (blog posts coming), but had a chance to mess around a bit with something that came up in an interview question this week.

I frequently ask candidates a high level design question to build PacMan. Games like pacman are fun because on the surface they are very simple, but if you don't structure your entities and their interactions correctly the design falls apart.

At some point during the interview we had scaled the question up such that there was now a problem of knowing at a particular point in the game what was nearby it. For example, if the board is 100000 x 100000 (10 billion elements) how efficiently can we determine if there is a nugget/wall next to us? One option is to store all of these entities in a 2d array and just access the neighbors. However, if the entity is any non trivial object, then we now have at [minumum 16 bytes](http://stackoverflow.com/a/258150/310196). That means we're storing 160 gigs to access the board. Probably not something we can realistically do on commodity hardware.

Given we're answering only a "is something there or not" question, one option is to bit pack the answer. In this sense you can leverage that each bit represents a coordinate in your grid. For example in a 2D grid

```
  
0 1  
2 3  

```

These positions could be represented by the binary value at that bit:

```
  
0 = 0b0001  
1 = 0b0010  
2 = 0b0100  
3 = 0b1000  

```

If we do that, and we store a list of longs (64 bits, 8 bytes) then to store 10 billion elements we need:

```
  
private val maxBits = maxX \* maxY  
private val requiredLongs = (maxBits / 64) + 1  

```

Which ends up being 22,032,273 longs, which in turn is 176.2 MB. Thats... a big savings. Considering that the trivial form we stored 10,000,000,000 objects, this is a compression ratio of 450%.

Now, one thing the candidate brought up (which is a great point) is that this makes working with the values much more difficult. The answer here is to provide a higher level API that hides away the hard bits.

I figured today I'd set down and do just that. We need to be able to do a few things

1. Find out how many longs to store
2. Find out given a coordinate which long it belongs to
3. In that long toggle the bit representing the coordinate if we want to set/unset it

```scala
  
class TwoDBinPacker(maxX: Int, maxY: Int) {  
 private val maxBits = maxX \* maxY  
 private val requiredLongs = (maxBits / 64) + 1  
 private val longArray = new Array[Long](requiredLongs)

def get(x: Int, y: Int): Boolean = {  
 longAtPosition(x, y).value == 1  
 }

def set(x: Int, y: Int, value: Boolean) = {  
 val p = longAtPosition(x, y)

longArray(p.index) = p.set(value)  
 }

private def longAtPosition(x: Int, y: Int): BitValue = {  
 val flattenedPosition = y \* maxX + x

val longAtPosition = flattenedPosition / 64

val bitAtPosition = flattenedPosition % 64

BitValue(longAtPosition, longArray(longAtPosition), bitAtPosition)  
 }  
}  

```

With the helper class of a BitValue looking like:

```scala
  
case class BitValue(index: Int, container: Long, bitNumber: Int) {  
 val value = (container \>\> bitNumber) & 1

def set(boolean: Boolean): Long = {  
 if (boolean) {  
 val maskAt = 1 \<\< bitNumber

container | maskAt  
 } else {  
 val maskAt = ~(1 \<\< bitNumber)

container & maskAt  
 }  
 }  
}  

```

At this point we can drive a scalatest:

```scala
  
"Bit packer" should "pack large sets (10 billion!)" in {  
 val packer = new TwoDBinPacker(100000, 100000)

packer.set(0, 0, true)  
 packer.set(200, 400, true)

assert(packer.get(0, 0))  
 assert(packer.get(200, 400))  
 assert(!packer.get(99999, 88888))  
}  

```

And this test runs in 80ms.

Now, this is a pretty naive way of doing things, since we are potentially storing tons of unused longs. A smarter way would be use a sparse set with skip lists, such that as you use a long you create it and mark it used, but things before it and after it (up to the next long) are marker blocks that can span many ranges. I.e.

```
  
{EmtpyBlock}[long, long, long]{EmptyBlock}[long]  

```

This way you don't have to store things you don't actually set.

Anyways, a fun little set of code to write. Full source available on my [github](https://github.com/devshorts/lru/blob/master/src/main/scala/com/devhorts/binpack/BinPacker.scala)

