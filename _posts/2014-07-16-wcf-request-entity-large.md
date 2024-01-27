---
layout: post
title: wcf Request Entity Too Large
date: 2014-07-16 20:17:06.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- wcf
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558691229;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4515;}}}}

permalink: "/2014/07/16/wcf-request-entity-large/"
---
I ran into a stupid issue today with WCF request entity too large errors. If you're sure your bindings are set properly on both the server and client, make sure to double check that the service name and contract's are set properly in the server.

My issue was that I had at some point refactored the namespaces where my service implementations were, and didn't update the web.config. For the longest time things continued to work, but once I reached the default max limit (even though I had a binding that set the limits much higher), I got the 413 errors.

So where I had this:

```
  
\<service name="Foo.Bar.Service"\>  
 \<endpoint address="" binding="basicHttpBinding" bindingConfiguration="LargeHttpBinding" contract="Foo.Bar.v1.Service.IService"/\>  
\</service\>  

```

I needed

```
  
\<service name="Foo.Bar.Implementation.Service"\>  
 \<endpoint address="" binding="basicHttpBinding" bindingConfiguration="LargeHttpBinding" contract="Foo.Bar.v1.Service.IService"/\>  
\</service\>  

```

How WCF managed to work when the service name was pointing to a non-existent class, I have no idea. But it did.

