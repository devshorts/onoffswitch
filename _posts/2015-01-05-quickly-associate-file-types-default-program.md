---
layout: post
title: Quickly associate file types with a default program
date: 2015-01-05 22:45:19.000000000 -08:00
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
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560873800;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4737;}i:1;a:1:{s:2:"id";i:3608;}i:2;a:1:{s:2:"id";i:2985;}}}}

permalink: "/2015/01/05/quickly-associate-file-types-default-program/"
---
I use JuJuEdit to open all my log files since it starts up fast, is pretty bare bones, but better than notepad. The way my log4net appender is set up is that log files are kept for 10 days and get a `.N` appended to them for each backup. I.e.

```
  
FooLog.log  
FooLog.log.1  
FooLog.log.2  

```

Etc.

I hate having to go through each one and set the default program to open since its slow and annoying. A faster way is to use cmd (not powershell!) and use the assoc and ftype commands.

You can associate an extension (like `.2`) with a "file type" (which doesn't really mean anything) and then map the file type to a program to open.

For example:

```
  
\>ftype logfile="C:\Program Files (x86)\Jujusoft\JujuEdit\JujuEdit.exe" %1  
\>assoc .3=logfile  
\>assoc .4=logfile  
\>assoc .5=logfile  
\>assoc .6=logfile  
...  

```

And now they all open with juju edit. If i ever want to change it I just re-run ftype and all my log files will now open with another program

