---
layout: post
title: Bad image format "Invalid access to memory location"
date: 2013-05-16 19:14:51.000000000 -07:00
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
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561330604;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3814;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4107;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/16/bad-image-format-invalid-access-memory-location/"
---
Wow, two bad image format posts in one day. So, the previous post talked about debugging 64bit vs 32 bit assemblies. But after that was solved I ran into another issue. This time with the message:

[csharp]  
Unhandled Exception: System.BadImageFormatException: Could not load file or assembly 'Interop.dll' or one of its dependencies. Invalid access to memory location. (Exception from HRESULT: 0x800703E6)  
 at Program.Program.Run(Args args, Boolean fastStart)  
 at Program.Program.Main(String[] args) in C:\Projects\Program.cs:line 36  
[/csharp]

Gah, what gives?

It seems that I had an interop DLL that was linking against pthreads. In debug mode, the dll worked fine on a 32 bit machine, but in release mode I'd get the error. The only difference I found was that in debug the dll was being explicity linked against pthreads lib file. Since pthread was also being built as a dynamic library, it's lib file contains information about functions and their address, but no actual code (that's in the dll).

After working for a while with my buddy [Faisal](http://fmansoor.wordpress.com/), we decided that even though the interop dll was being built and linked properly, when it wasn't being explicity linked to the lib file the function addresses were somehow wrong. What the compiler was actually linking against we never figured out. But, it does all make sense. If the function addresses were wrong, then when it would try and access anything in the dll it could access memory outside of its space and get an exception.

Once we explicitly linked the library as part of the linker settings of the interop dll everything worked fine.

