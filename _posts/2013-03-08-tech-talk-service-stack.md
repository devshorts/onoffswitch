---
layout: post
title: 'Tech talk: Service stack'
date: 2013-03-08 16:25:48.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- servicestack
meta:
  _edit_last: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560613195;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:3696;}i:2;a:1:{s:2:"id";i:4939;}}}}

permalink: "/2013/03/08/tech-talk-service-stack/"
---
Today's tech talk the team and I talked about [ServiceStack](http://www.servicestack.net/). I've heard a lot of hype about it but never really understood what it did or was about. Today, unfortunately, didn't really clear any of that up.

We watched the ServiceStack powerpoint on their site and went to their [github](https://github.com/ServiceStack/ServiceStack) to look at the examples. What I got out of all of that is that ServiceStack is trying to simplify the creation of service oriented architectures (SOA) by providing a complete stack architecture for you to use (an ORM, backing cache/NoSql, routes, serialization, etc).

The confusion, however, lays in the fact that most other solutions are doing nearly the same thing. MVC has routes and web-api's with GET/POST/PUT/DELETE methods. WCF exposes transport independent RPC. Node.js has routes with auto fill parameters. Everyone is using JSON serialization. Redis is common as a caching layer. A bunch of ORM's already exist for SQL. Enforcement of lightweight DTO's. I couldn't really tell what made ServiceStack so much better than anything else.

ServiceStack boasts of a faster ORM then NHibernate and Entity Framework, as well as the fastest .NET JSON serializer, but the comparison charts were almost 3 years old (a lifetime in technology!). I did see that ServiceStack was less verbose to get set up making a web-api than MVC4, but the code looked pretty similar. If you rolled an app using any ORM, MVC or WCF, created your own DTO's, and sent everything over JSON then I think you basically have ServiceStack? Frankly I'm as confused as ever.

The team, I think, feels the same way I do. We didn't have a very spirited conversation because everyone was a little lost as to what the significance of ServiceStack was. We came out of it thinking that maybe to appreciate ServiceStack's offerings better that we should have a hack day to build a small service oriented application in node.js, MVC4, IIS hosted WCF, and ServiceStack.

But, from my few hours of looking at the examples I'm not really sold on what is so great about it.

