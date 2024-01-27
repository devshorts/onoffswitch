---
layout: post
title: SignalR on ios and a single domain
date: 2013-07-22 08:00:21.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- ios
- realtime
- SignalR
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1555086534;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3392;}i:1;a:1:{s:2:"id";i:289;}i:2;a:1:{s:2:"id";i:2365;}}}}

permalink: "/2013/07/22/signalr-ios-single-domain/"
---
Safari on ios has a limitation that you can only have one concurrent request to a particular domain at a time. Normally this is fine, since once a request completes the next one that is queued up fires off. But what if you are using a realtime persistent connection library like signalR? In this case your one allowed connection is held up with the signalR request. If you're not on a mac or linux and you use windows 7 or earlier you can't use websockets so you're stuck using http. Most suggestions involve buying a second domain, but sometimes thats not possible, especially if your application is a distributable web app that can run on client machines. You can't expect clients to have to buy a second domain just so your realtime push works.

A nice solution, posted about in [this github tracker](https://github.com/SignalR/SignalR/issues/1406) following the issue is to configure your signalR's long poll mechanism to work in short bursts. Instead of having a single persistent connection, you can tell the client to use the long poll transport (using typescript here)

[ts]  
var signalRConfig:def.IHubConfiguration = commonUtils.MiscUtil.userIsIphone() ?  
 {  
 transport: "longPolling"  
 } : {};

// we can't do signalR push with an iphone since  
// iphones only allow for single connections, so having the  
// forever frame interferes with other resource loading  
$.connection.hub.start(signalRConfig, () =\> console.log("connected"));  
[/ts]

And on the server, in Global.asax.cs, configure the long poll bursts

```csharp
  
protected void Application\_Start(object sender, EventArgs e)  
{  
 GlobalHost.Configuration.ConnectionTimeout = TimeSpan.FromMilliseconds(1000);  
 LongPollingTransport.LongPollDelay = 5000;  
 RouteTable.Routes.MapHubs();  
}  

```

But when I did this, per the tracker suggestion, I saw that I was still getting bursts that lasted 5 seconds. This was too long for me, since for 5 seconds you can't do anything else. If you are doing a lot of dynamic calls (making AJAX requests, or other service calls) then 5 seconds really holds up the application.

When in doubt, read the source. I'm unfortunately using a really old version of signalR because I'm tied to that version due to how my application is distributed (and that signalR versions aren't backwards compatible), so this advice may not apply to later versions. But, I found what the issue was.

SignalR uses a heartbeat to check when things are disconnected or timed out:

```csharp
  
\_timer = new Timer(Beat,  
 null,  
 \_configurationManager.HeartBeatInterval,  
 \_configurationManager.HeartBeatInterval);

//....

private void Beat(object state)  
{  
 //...  
 foreach (var metadata in \_connections.Values)  
 {  
 if (metadata.Connection.IsAlive)  
 {  
 CheckTimeoutAndKeepAlive(metadata);  
 }  
 else  
 {  
 // Check if we need to disconnect this connection  
 CheckDisconnect(metadata);  
 }  
 }  
 /...  
}  

```

The issue here is that the heartbeat is initialized to 10 seconds

```csharp
  
public DefaultConfigurationManager()  
{  
 ConnectionTimeout = TimeSpan.FromSeconds(110);  
 DisconnectTimeout = TimeSpan.FromSeconds(20);  
 HeartBeatInterval = TimeSpan.FromSeconds(10);  
 KeepAlive = TimeSpan.FromSeconds(30);  
}  

```

So, that means that no matter what you set the disconnect and connection timeouts to be, they can't be any more granular than 10 seconds. If you remember from signal processing the [shannon-nyquist sampling theorum](http://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem)

![](http://onoffswitch.net/wp-content/uploads/2013/07/953b4b6e51335f67619cad644c437858.png)

You know that your sampling frequency needs to be twice the target frequency. Fancy words for sample more to get more granular

The final fix I needed to add was

[csharp highlight="4"]  
protected void Application\_Start(object sender, EventArgs e)  
{  
 GlobalHost.Configuration.ConnectionTimeout = TimeSpan.FromMilliseconds(1000);  
 GlobalHost.Configuration.HeartBeatInterval = TimeSpan.FromSeconds(GlobalHost.Configuration.ConnectionTimeout.TotalSeconds/2);  
 LongPollingTransport.LongPollDelay = 5000;  
 RouteTable.Routes.MapHubs();  
}  
[/csharp]

Now that the heartbeat interval is twice as fast as the connection timeout, I properly get 1 second bursts followed by 5 seconds of down time:

![2013-07-09 11_27_08-Charles 3](http://onoffswitch.net/wp-content/uploads/2013/07/2013-07-09-11_27_08-Charles-3.png)

