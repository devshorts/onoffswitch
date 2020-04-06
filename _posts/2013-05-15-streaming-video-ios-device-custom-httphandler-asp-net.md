---
layout: post
title: Streaming video to ios device with custom httphandler in asp.net
date: 2013-05-15 21:44:20.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- asp.net
- http
- iphone
- video
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _wp_old_slug: streaming-h-264-video-ipad-iphone-custom-http-handler-asp-net
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561920465;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4028;}i:1;a:1:{s:2:"id";i:1587;}i:2;a:1:{s:2:"id";i:3524;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/15/streaming-video-ios-device-custom-httphandler-asp-net/"
---
I ran into an interesting tidbit just now while trying to dynamically stream a video file using a custom http handler. The idea here is to bypass the static handler for a file so that I can perform authentication/preprocessing/etc when a user requests a video resource and I don't have to expose a static folder with potentially sensitive resources.

I had everything working fine on my desktop browser, but when I went to test on my iPhone I got the dreaded play button with a circle crossed out

[![noVideo](http://onoffswitch.net/wp-content/uploads/2013/05/noVideo.png)](http://onoffswitch.net/wp-content/uploads/2013/05/noVideo.png)

I hate that thing.

Anyways, streaming a file from the static handler worked fine though, so what was the difference? This is where I pulled out charles and checked the response headers.

From the static handler I'd get this:

[code]  
HTTP/1.1 200 OK  
Content-Type video/mp4  
Last-Modified Wed, 15 May 2013 20:59:30 GMT  
Accept-Ranges bytes  
ETag "9077fe17af51ce1:0"  
Server Microsoft-IIS/7.5  
X-Powered-By ASP.NET  
Date Wed, 15 May 2013 21:23:41 GMT  
Content-Length 2509720  
[/code]

And for my dynamic handler I got this:

[code]  
HTTP/1.1 200 OK  
Cache-Control private  
Content-Type video/mp4  
Server Microsoft-IIS/7.5  
X-AspNet-Version 4.0.30319  
X-Powered-By ASP.NET  
Date Wed, 15 May 2013 21:22:55 GMT  
Content-Length 2509720  
[/code]

So all that was missing was ETag and Accept-Ranges.

Turns out ETag is a good thing to have since its a CRC of the file. It tells the client if anything has actually changed or not and helps with caching. Not a bad thing to have, especially if you use a fast hashing algorithm to generate your signature.

The second thing that was different was the Accept-Ranges header. This turns out to be a way for the client to make certain range requests in case a connection is closed or something fails. From the [spec](http://greenbytes.de/tech/webdav/draft-ietf-httpbis-p5-range-latest.html):

> Hypertext Transfer Protocol (HTTP) clients often encounter interrupted data transfers as a result of canceled requests or dropped connections. When a client has stored a partial representation, it is desirable to request the remainder of that representation in a subsequent request rather than transfer the entire representation. **Likewise, devices with limited local storage might benefit from being able to request only a subset of a larger representation, such as a single page of a very large document, or the dimensions of an embedded image**.
> 
> This document defines HTTP/1.1 range requests, partial responses, and the multipart/byteranges media type. Range requests are an optional feature of HTTP, designed so that recipients not implementing this feature (or not supporting it for the target resource) can respond as if it is a normal GET request without impacting interoperability. Partial responses are indicated by a distinct status code to not be mistaken for full responses by caches that might not implement the feature.
> 
> Although the range request mechanism is designed to allow for extensible range types, this specification only defines requests for byte ranges.

However, it's up the server to tell the client that it supports range requests. So, once I added this into the header

[csharp]  
context.Response.Headers.Add("Accept-Ranges", "bytes");  
context.Response.Headers.Add("ETag", HashUtil.QuickComputeHash(target));  
context.Response.ContentType = "video/mp4";  
context.Response.TransmitFile(target);  
[/csharp]

Everything started to work. Now, I'm not actually handling range requests though. I need to test with a huge file and kill the connection and see what happens. But, even then, all that requires is [testing the request headers](http://stackoverflow.com/questions/4330023/detecting-byte-range-requests-in-net-httphandler) and parsing for the byte range it wants to send out.

Turns out I'm not the only one to see this happen. Check out [here](http://www.markeverard.com/2011/07/05/serving-videos-to-ios-devices-from-episerver-vpp-folders/) and [here](http://dotnetslackers.com/articles/aspnet/Range-Specific-Requests-in-ASP-NET.aspx) for more info.

