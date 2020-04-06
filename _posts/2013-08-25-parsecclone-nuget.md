---
layout: post
title: ParsecClone on nuget
date: 2013-08-25 16:10:04.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- combinator
- nuget
- parser
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554986094;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4131;}i:1;a:1:{s:2:"id";i:2735;}i:2;a:1:{s:2:"id";i:4077;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/08/25/parsecclone-nuget/"
---
Today I published the first version of [ParsecClone](https://github.com/devshorts/ParsecClone) to [nuget](https://www.nuget.org/packages/ParsecClone/). I [blogged](http://onoffswitch.net/parsing-csvs-parser-combinator/) recently about creating my own parser combinator and it's come along pretty well. While [FParsec](http://www.quanttec.com/fparsec/) is more performant and better optimized, mine has other advantages (such as being able to work on arbitrary consumption streams such as binary or bit level) and work directly on strings with regex instead of character by character. Though I wouldn't recommend using ParsecClone for production string parsing if you have big data sets, since the string parsing isn't streamed. It works directly on a string. That's still on the todo list, however the binary parsing does work on streams.

Things included:

- All your favorite parsec style operators: ``, `>>.`, `.>>`, `|>>`, etc. I won't list them all since there are a lot.
- String parsing. Match on full string terms, do regular expression parsing, inverted regular expressions, etc. I have a full working CSV parser written in ParsecClone
- Binary parsing. Do byte level parsing with endianness conversion for reading byte arrays, floats, ints, unsigned ints, longs, etc. 
- Bit level parsing. Capture a byte array from the byte parsing stream and then reprocess it with bit level parsing. Extract any bit, fold bits to numbers, get a list of zero and ones representing the bits you captured. Works for any size byte array (though converting to int will only work for up to 32 bit captures).

The fun thing about ParsecClone is you can now parse anything you want as long as you create a streamable container. The combinator libraries don't care what they are consuming, just that they are combining and consuming. This made it easy to support strings, bytes, and bits, all as separate consumption containers.

Anyways, maybe someone will find it useful, as I don't think there are any binary combinator libraries out there for F# other than this one. I'd love to get feedback if anyone does use it!

