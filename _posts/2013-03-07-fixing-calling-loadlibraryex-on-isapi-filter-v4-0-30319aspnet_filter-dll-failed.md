---
layout: post
title: Fixing "Calling LoadLibraryEx on ISAPI filter v4.0.30319 aspnet_filter.dll
  failed"
date: 2013-03-07 16:22:10.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- 64 bit
- iis
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561366663;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3814;}i:1;a:1:{s:2:"id";i:4914;}i:2;a:1:{s:2:"id";i:4028;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/07/fixing-calling-loadlibraryex-on-isapi-filter-v4-0-30319aspnet_filter-dll-failed/"
---
[code wraplines="true"]Calling LoadLibraryEx on ISAPI filter "C:\Windows\Microsoft.NET\Framework\v4.0.30319\\aspnet\_filter.dll" failed[/code]

I ran into this adding a new site to my 64 bit machine, and because I haven't had my morning coffee, I forgot to set "enable 32 bit applications" in the app pool.

If your code is built for 32 bit only (maybe you use mixed mode dll's somewhere or call into native and can't be 64 bit for whatever reason), make sure the app pool of your application is set to 32 bit mode. Otherwise, you get the very descriptive error shown above.

![iis32bit.](http://onoffswitch.net/wp-content/uploads/2013/03/iis32bit.-600x434.png)

