---
layout: post
title: Multiple SignalR clients and ASMX service calls from the same application
date: 2014-07-07 08:00:47.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- SignalR
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1556647018;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3392;}i:1;a:1:{s:2:"id";i:4091;}i:2;a:1:{s:2:"id";i:2365;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2014/07/07/multiple-signalr-clients-asmx-service-calls-application/"
---
I was writing a test application to simulate what multiple signalR clients to a server would act like. The clients were triggered by the server and then would initiate a sequence of asmx web service calls back to the server using a legacy web service. This way I was using signalR as a triggering mechanism and not as a data transport. For my purpose this worked out great.

I had coupled the asmx calling code into a test class for a signalR client, so each class was responsible for its internal signalR connection as well as its outgoing asmx calls. When I had one class everything worked great. But the moment I had two classes running (i.e 2 signalR connections and 2+ asmx connections) everything locked up. I couldn't figure out what was going on. The signalR clients had connected but all the asmx calls stopped making it through, and eventually I got errors like this:

[csharp]  
System.Net.WebException: The operation has timed out  
at System.Net.HttpWebRequest.GetRequestStream(TransportContext& context)  
at System.Net.HttpWebRequest.GetRequestStream()  
at System.Web.Services.Protocols.SoapHttpClientProtocol.Invoke(String methodName, Object[] parameters)  
[/csharp]

At first I thought maybe the server was rejecting calls or was overloaded somehow. It didn't make any sense since it was only 4 connections, but I was testing against a server with other live connections so I figured I'd rule that option out. But when I looked at the IIS logs of the server I didn't even see any incoming connections. No 503 errors were generated, nothing indicated that the calls were even making it outbound from the client.

I decided to step back a little and took out the signalR client code and used an Rx observable timer to fire off asmx calls every 100 milliseconds in multiple threads trying to simulate a server load but everything worked fine. The moment I tied signalR back in with more than 1 client everything stopped.

After that I started to get curious if there was some sort of outbound connection limit. I spawned two instances of my test application, each with a single signalR client and the asmx services firing, and each app worked. So whatever was going on was definitely process bound, and not operating system bound.

It took a lot of reading about IIS settings and other blog entries to finally find what I was looking for. Turns out that each outgoing http call is handled by a [`ServicePoint`](http://msdn.microsoft.com/en-us/library/system.net.servicepoint.aspx) class. From msdn:

> The ServicePoint class handles connections to an Internet resource based on the host information passed in the resource's Uniform Resource Identifier (URI). The initial connection to the resource determines the information that the ServicePoint object maintains, which is then shared by all subsequent requests to that resource.
> 
> ServicePoint objects are managed by the ServicePointManager class and are created, if necessary, by the ServicePointManager.FindServicePoint method. ServicePoint objects are never created directly but are always created and managed by the ServicePointManager class. The maximum number of ServicePoint objects that can be created is set by the ServicePointManager.MaxServicePoints property.

In the `ServicePointManager` there is a property called `DefaultConnectionLimit` which had this tidbit of a comment

> The maximum number of concurrent connections allowed by a ServicePoint object. The default value is 2.

Turns out since I was connecting all the signalR clients and the asmx calls through the same server uri it was all using the same ServicePoint object which limited the connections to 2. When I ran just the asmx calls they didn't block because they were short bursts, when one completed the next one in the queue was handled, but when I had the two persistent signalR http connections I had used all the available resources and nothing would complete.

I added

[csharp]ServicePointManager.DefaultConnectionLimit = 10;[/csharp]

to the start of my application and everything worked out great.

Why they've limited the connection to 2 I'm not sure. I think its limited to minimize target server loads and optimize the usual fire and forget use cases, though with more people going towards long polling and persistent connections I think this will come up more often.

For more reading check out [this great post](http://blogs.msdn.com/b/jpsanders/archive/2009/05/20/understanding-maxservicepointidletime-and-defaultconnectionlimit.aspx) about default connection limit and max service point idle time property settings.

