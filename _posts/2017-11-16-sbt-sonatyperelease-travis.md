---
layout: post
title: Sbt sonatypeRelease on Travis
date: 2017-11-16 18:52:01.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- sbt
- travis
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"37550b67d263a3ce789993dc25046c5f";a:2:{s:7:"expires";i:1557076392;s:7:"payload";a:6:{i:0;a:1:{s:2:"id";i:4892;}i:1;a:1:{s:2:"id";i:4939;}i:2;a:1:{s:2:"id";i:4991;}i:3;a:1:{s:2:"id";i:4737;}i:4;a:1:{s:2:"id";i:4327;}i:5;a:1:{s:2:"id";i:4191;}}}}
  _su_rich_snippet_type: none

permalink: "/2017/11/16/sbt-sonatyperelease-travis/"
---
I figured I'd drop a quick note here for anyone else running into an issue. If you are trying to do a sonatypeRelease via sbt 1.0.3 on travis and getting a

```text
  
Credentials file /home/travis/.sbt/credentials does not exist  

```

Even though you are supplying your own inline creds, just drop in a fake creds file into that location:

```text
  
realm=Sonatype Nexus Repository Manager  
host=x.y.z  
user=none  
password=none  

```

Via

```text
  
cp scripts/creds.fake /home/travis/.sbt/credentials  

```

And save yourself the 2 days of headache I have had :p

