---
layout: post
title: Seq.unfold and creating bit masks
date: 2013-09-09 20:28:22.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- bits
- byte
- F#
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559923232;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4213;}i:1;a:1:{s:2:"id";i:3615;}i:2;a:1:{s:2:"id";i:4286;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/09/09/seq-unfold-creating-bit-masks/"
---
In the course of working on [ParsecClone](https://github.com/devshorts/ParsecClone) I needed some code that could take in an arbitrary byte array and convert it to a corresponding bit array. The idea is if I have an array of

[fsharp]  
[|byte 0xFF;byte 0x01|]  
[/fsharp]

Then I should get

[fsharp]  
[|1;1;1;1;1;1;1;0;0;0;0;0;0;0;1|]  
[/fsharp]

I've done plenty of bit slingin' in my day, and the trick is just to apply a sequence of bit masks to each byte and collect all the bit arrays. In other languages this is always a little bit of a pain, but in F# the solution was amazingly elegant

## Data

As with anything F#, I like to start with the data

[fsharp]  
type Bit =  
 | One  
 | Zero  
 override this.ToString() =  
 match this with  
 | One -\> "1"  
 | Zero -\> "0"  
[/fsharp]

## Make bit masks

Now lets generate the meat and potatoes of this: a sequence of bit masks

[fsharp]  
let bitMasks = Seq.unfold (fun bitIndex -\> Some((byte(pown 2 bitIndex), bitIndex), bitIndex + 1)) 0  
 |\> Seq.take 8  
 |\> Seq.toList  
 |\> List.rev  
[/fsharp]

While a `fold` takes a list and a seed and returns a single accumulated item, an `unfold` takes a seed and generates a list. For those not familiar, `unfold` takes a function of the signature

[fsharp]  
(State -\> ('a \* State) option) -\> State -\> Seq\<'a\>  
[/fsharp]

Unfold takes a function with an argument that is the state, and returns an `item * state` option tuple. The first element of the option is the element to be emitted in the sequence. The second item is the _next_ state. If you return `None` instead of `Some` the infinite sequence will end. You can see that my state is the exponent n of _2^n_ which gives you the bit mask. The first iteration is 2^0, then 2^1, then 2^2, etc. By reversing it, I now have a bitmask that look like this:

[fsharp]  
[2^7; 2^6; 2^5; 2^4; 2^3; 2^2; 2^1; 2^0]  
[/fsharp]

## Byte to Bits

The next thing is to apply the bitmask to a byte.

[fsharp]  
let byteToBitArray b =  
 List.map (fun (bitMask, bitPosition) -\>  
 if (b &&& bitMask) \>\>\> bitPosition = byte(0) then Zero  
 else One) bitMasks  
[/fsharp]

The unusual thing here is that bitwise and is the `&&&` operator and bitwise shift is the \>\>\> operator. Not that weird, but different from other langauges.

## Bytes to Bits

All that's left is applying the byteToBitArray function to byte array to get a bit array

[fsharp]  
let bytesToBits (bytes:byte[]) =  
 bytes  
 |\> Array.toList  
 |\> List.map byteToBitArray  
 |\> List.collect id  
 |\> List.toArray  
[/fsharp]

And now to test it in fsi

[fsharp]  
\> bytesToBits [|byte 0xFF;byte 0x01|];;  
val it : Bit [] =  
 [|One; One; One; One; One; One; One; One; Zero; Zero; Zero; Zero; Zero; Zero;  
 Zero; One|]  
[/fsharp]

## Bits To UInt

We can even take a bit array and create a uint now too

[fsharp]  
let bitsToUInt (bits:Bit[]) =  
 let positions = Array.zip bits (Array.rev [|0..Array.length bits - 1|])

Array.fold (fun acc (bit, index) -\>  
 match bit with  
 | Zero -\> acc  
 | One -\> acc + (pown 2 index)) 0 positions  
[/fsharp]

First I zip the bit array with each position in the bit array. Then we just need to fold over the array and add the accumulator to _2^n_ if the bit is a one.

[fsharp]  
\> [|One; One; One; One; One; One; One; Zero;|] |\> bitsToUInt;;  
val it : int = 254  
[/fsharp]

## Conclusion

I really enjoyed working with the higher order functions that F# provides to make a simple and robust conversion. Working with strongly typed data felt more robust than dealing with just integers.

