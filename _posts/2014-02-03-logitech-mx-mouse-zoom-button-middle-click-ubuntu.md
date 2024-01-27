---
layout: post
title: Logitech mx mouse zoom button middle click on Ubuntu
date: 2014-02-03 03:58:32.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags: []
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560926272;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4244;}i:1;a:1:{s:2:"id";i:4905;}i:2;a:1:{s:2:"id";i:4306;}}}}

permalink: "/2014/02/03/logitech-mx-mouse-zoom-button-middle-click-ubuntu/"
---
Any good engineer has their own tools of their trade: keyboard, mouse, and licenses to their favorite editors (oh and a badass [chair](http://www.hermanmiller.com/products/seating/performance-work-chairs/embody-chairs.html)).

I work now on an Ubuntu box and I wanted to get my logitech MX mouse's zoom button to act as middle click. I really like this functionality since its easy to copy, paste, close windows, and open new links with this button.

However, the button mapping in Ubuntu isn't trivial. On windows you used the setpoint program to do it and called it a day. But in linux land you need to put more work into it.

[Here](http://forums.logitech.com/t5/Mice-and-Pointing-Devices/Guide-for-setup-Performance-MX-mouse-on-Linux-with-KDE/td-p/517167) is a great tutorial describing how to do it, but for the lazy, here is the mapping you need.

```
  
"xte 'mouseclick 2'"  
 b:13+Release  

```

What this says is "when button 13 is clicked, then released, issue a mouseclick 2 command". `xte` is a program that simulates mouse and keyboard events, and `xbindkeys` (whose config you edit to set the xte mapping) is a program that lets you bind one key or mouse event to another key or mouse event.

Once I did this and started up `xbindkeys` then my zoom button (button 13) now worked as middle click (mouseclick 2).

