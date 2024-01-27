---
layout: post
title: Dealing with a bad symbolic reference in scala
date: 2016-07-25 22:55:06.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- dependencies
- maven
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554608701;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4939;}i:1;a:1:{s:2:"id";i:4919;}i:2;a:1:{s:2:"id";i:4028;}}}}
  _wpas_done_all: '1'

permalink: "/2016/07/25/dealing-bad-symbolic-reference-scala/"
---
Every time this hits me I have to think about it. The compiler barfs at you with something ambiguous like

> [error] error: bad symbolic reference. the classpath might be incompatible with the version used when compiling Foo.class.

What this really is saying is that `Foo.class` references some import or class whose namespace isn't on the classpath or has fields missing. I usually get this when I have a project level circular dependency via transitive includes. I.e.

```
  
Repo 1/  
 /project A  
 /project B -\> depends on C and A  
Repo 2  
 /project C -\> depends on A  

```

So here the dependency `C` pulls in a version of `A` but that version may not be the same that project `B` pulls in. If I do namespace refactoring in project A, then project B won't compile if those namespaces are used by project C. It's a mess.

Thankfully scala lets you maintain folder structure outside of package namespace, unlike java. So I can fake it till I make it by refactoring and keeping the old namespace, until I get a working build and then updating the secondary repo. It's like a two phase commit.

