---
layout: post
title: Command pattern with SignalR
date: 2013-04-15 08:00:17.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- design patterns
- json
- SignalR
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561935122;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4091;}i:1;a:1:{s:2:"id";i:4116;}i:2;a:1:{s:2:"id";i:289;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/04/15/command-pattern-with-signalr/"
---
I'm using SignalR as a long poll mechanism between multiple .NET clients because part of my projects requirements is to have everything over http/https. There's no point in rolling my own socket based long poll since SignalR has already done all the heavy lifting. Unfortunately since the app I work on is distributed I can't upgrade my SignalR version from what I have (0.5.2) since the newer SignalR versions aren't backwards compatabile. This means I have to make do with what this version of SignalR gives me.

Still, the version I have works great for my purposes and I can't complain. However, I was getting frustrated at having to constantly update my internal boilerplate whenever I wanted to add a new hub method dispatch. What I want is to have a single method dispatch and encapsulate all my business logic in a command pattern so that I never have to update my SignalR hub code. Instead I want to only need to add a new command and everything magically gets executed when someone sends a remote action using it.

## The Old Way

As an example, you can (or used to, since I know SignalR has changed a lot since the version that I have), do this to create a hub proxy connection:

[csharp]  
var Connection = new HubConnection(Url);

var Hub = Connection.CreateProxy("hubName");

Hub.On\<string\>("DynamicMethodCall", DynamicMethodReference);  
[/csharp]

So when the server invokes

[csharp]  
client.DynamicMethodCall("foo");  
[/csharp]

Your client will get `foo` passed to the `DynamicMethodReference` function.

But if you start having a lot of client method calls, then this becomes a pain to update. Each time you want to dispatch a new method you have to update both your client and your server and its easy to forget to do. You may be thinking "_but its just one server call and one client call, so what, who cares?_", and I agree, if you have a simple use case.

Of course nothing is ever simple, right? In my application, I have a facade hiding the actual SignalR implementation. The outside code doesn't touch the dynamic clients object, since there is a lot of code to manage finding the right client based on a bunch of things like separate keys and who is and isn't connected, etc.

In addition, when you work in a large application it's always a good idea to separate out 3rd party libraries behind firewalls like that, it makes them easier to update later. The last thing you want is a spaghetti mess of 3rd party code all over your app that you can't update easily. It's a little more work up front but it pays off later. The downside is that to make changes to the external API you need to also update the internals. So, being able to have one single entry point and exit point makes things a lot neater in the long run. Less boilerplate means more fun times making things do stuff!

## Commands

To solve this problem, I wanted to send an [encapsulated object](http://en.wikipedia.org/wiki/Command_pattern) that contained my data and code to do the work I wanted. For example, what I wanted to end up with was:

[csharp]  
var Connection = new HubConnection(Url);

var Hub = Connection.CreateProxy("hubName");

Hub.On\<RemoteCommand\>("RemoteCommand", RemoteCommadHandler);  
[/csharp]

Where `RemoteCommand` would be an abstract base class. This would let me focus on my business logic and not my interconnect logic. If I wanted to invoke an echo on a client I could do this:

[csharp]  
public class EchoCommand : RemoteCommand  
{  
 public override void Execute(){  
 Console.WriteLine("Echoecho!");  
 }  
}  
[/csharp]

And to send it from the server would look like this

[csharp]  
client.RemoteCommand(new EchoCommand());  
[/csharp]

Later, if I had something else I wanted to invoke all I'd need to do is make a new command and dispatch it on the client again.

[csharp]  
public class LogMeCommand : RemoteCommand  
{  
 public override void Execute(){  
 Log.Debug("log me happened!");  
 }  
}  
[/csharp]

[csharp]  
client.RemoteCommand(new LogMeCommand ());  
[/csharp]

## But...

I was hoping this would work but originally it didn't. Obviously the fact that I had registered an abstract base class was preventing it from working.

The first thing I did was change my `Hub.On` code to be of type `dynamic`. When I did that I was able to execute the registered command, but it came back as a json type. This tipped me off that the default JSON serializer that SignalR was using didn't serialize the inheritance structure.

To test this theory out I manually serialized json on the server side, sent a string over the wire, then deserialized the item on the client side. This worked perfectly and maintained all my inheritance structure.

Next step was to find out how to replace the JSON serializer in an old SignalR version. The SignalR dev's theoretically made SignalR extremely extensible. They've exposed a dependency injection hook to register your own serializer. All they claim you had to do was add

[csharp]  
GlobalHost.DependencyResolver.Register(  
 typeof (IJsonSerializer),  
 () =\> JsonSerializer.Create(new JsonSerializerSettings  
 {  
 TypeNameHandling = TypeNameHandling.All  
 }));  
[/csharp]

In the `Application_Start` function of your site (or wherever your app start may be). But, this didn't work for me. Not sure what I was doing wrong, but my endpoints that needed to serialize this base class never got hit. Other endpoints that had basic types for the hub argument worked fine.

## The Solution

Eventually I settled back on my hacky way of doing it, but at least the hack is hidden in my boilerplate and I wont have to come back to it.

When sending out I do

[csharp]  
var serializer = new JsonSerializerSettings { TypeNameHandling = TypeNameHandling.All };  
var str = JsonConvert.SerializeObject(command, serializer);  
client.RemoteCommand(str);  
[/csharp]

And when receiving I do

[csharp]  
Hub.On\<string\>("RemoteCommand", item =\>  
 {  
 var rcmd = JsonConvert.DeserializeObject\<RemoteCommand\>(  
 item,  
 new JsonSerializerSettings  
 {  
 TypeNameHandling = TypeNameHandling.All  
 });  
 RemoteCommandRequest(rcmd);  
 });  
[/csharp]

And now everything works great.

