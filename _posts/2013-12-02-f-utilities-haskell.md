---
layout: post
title: F# utilities in haskell
date: 2013-12-02 19:52:20.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- F#
- haskell
- Utilities
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554338514;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4262;}i:1;a:1:{s:2:"id";i:4244;}i:2;a:1:{s:2:"id";i:4725;}}}}

permalink: "/2013/12/02/f-utilities-haskell/"
---
Slowly I am getting more familiar with Haskell, but there are some things that really irk me. For example, a lot of the point free functions are right to left, instead of left to right. Coming from an F# background this drives me nuts. I want to see what happens first _first_ not _last_.

For example, if we wanted to do `(x+2)+3` in f#

[fsharp]  
let chained = (+) 2 \>\> (+) 3  
[/fsharp]

Compare to haskell:

[haskell]  
chained :: Integer -\> Integer  
chained = (+3) . (+2)  
[/haskell]

In haskell, the +2 is given the argument 3, then that value is given to +3. In f# you work left to right, which I think is more readable.

Anyways, this is an easy problem to solve by defining a cusotm infix operator

[haskell]  
(\>\>\>) :: (a -\> b) -\> (b -\> c) -\> (a -\> c)  
(\>\>\>) a b = b . a  
[/haskell]

Now we can do the same combinations as in F#.

Another thing that bugs me is pipe operator in haskell. I want to be able to pipe using the `|>` operator left to right (as in subject followed by verb) instead of the way haskell does it with `$`which is verb followed by subject.

Again, easy fix though

[haskell]  
(|\>) :: t -\> (t -\> b) -\> b  
(|\>) a b = b $ a  
[/haskell]

Now we can do `1 |> (+1)` and get `2`. Fun!

