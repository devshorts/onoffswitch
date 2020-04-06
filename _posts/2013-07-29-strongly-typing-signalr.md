---
layout: post
title: Strongly typing SignalR
date: 2013-07-29 08:00:43.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dynamic proxy
- SignalR
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560802337;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3392;}i:1;a:1:{s:2:"id";i:289;}i:2;a:1:{s:2:"id";i:4091;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/07/29/strongly-typing-signalr/"
---
I'm a big fan of strong typing. If you can leverage the compiler to give you an error (or warning) before you deploy code, all the better. That means you won't, ideally, push a bug into the field. So I have a big problem with frameworks and libraries that rely on dynamic objects, or even worse, stringly typing thing. Don't get me wrong, sometimes dynamics are the only way to solve the problem, but whenever I run into one I'm always afraid that I'm going to get a runtime error since I don't _really_ know what I'm acting on till later.

In this post, I'm going to discuss strongly typing signalR. For the impatient, I have a [working demo up](http://strongsignalr.apphb.com/), as well as the code posted on my [github](https://github.com/devshorts/StronglyTypedSignalr).

That said, I've written about signalR before so I won't rehash that, but signalR uses dynamic objects heavily to give you the flexibility of "invoking" whatever method you want on the client side. In my first forays using signalR I went with this iconic chat example:

[csharp]  
public void Send(string message)  
{  
 // Call the addMessage method on all clients  
 Clients.All.addMessage(message);  
}  
[/csharp]

`Clients.All` is a dynamic object, and `addMessage` is going to be a registered handler in the javascript side. Because it's dynamic, a small typo can cause your client side invocation to never succeed. You won't get an error, just nothing will happen. That's almost even worse than getting an exception!

But, if we know a little about the signalR internals (which we can since signalR is open source), we can solve all these issues with almost no extra code.

First, lets start with defining what we want to do:

[csharp]  
public interface IJsMethods  
{  
 void PrintString(string msg);  
}  
[/csharp]

We'll say that "PrintString" is an available javascript method to call and it has some specific arguments to use. Inside of our signalR hub, the goal is going to be to be able to do this:

[csharp]  
AllClients.PrintString("Everyone gets the time! " + DateTime.Now.ToString())  
[/csharp]

Which should invoke a `printString` method in javascript with a string parameter.

If we change the interface later, we should get compile time errors and we can be confident that we'll be invoking the right things on the client.

Back to knowing a little about the signalR internals. If you inspect the type of `Clients.All` (or look at the signalR source), you'll see that it actually resolves at runtime to be of type `ClientProxy` which implements `IClientProxy`. This makes our lives pretty easy, since we can write an interceptor for `IClientProxy` and do the invocation of the client side javascript for us.

[csharp]  
public static class HubExtensions  
{  
 private static readonly ProxyGenerator Generator = new ProxyGenerator();

public static T AsStrongHub\<T\>(this IClientProxy source)  
 {  
 return (T)Generator.CreateInterfaceProxyWithoutTarget(typeof(T), new StrongClientProxy(source));  
 }  
}

public class StrongClientProxy : IInterceptor  
{  
 public IClientProxy Source { get; set; }

public StrongClientProxy(IClientProxy source)  
 {  
 Source = source;  
 }

public void Intercept(IInvocation invocation)  
 {  
 var methodName = StringUtil.FirstLower(invocation.Method.Name);

Source.Invoke(methodName, invocation.Arguments);  
 }  
}  
[/csharp]

And we can call this from our hub using:

[csharp]  
private IJsMethods AllClients  
{  
 get { return (Clients.All as ClientProxy).AsStrongHub\<IJsMethods\>(); }  
}  
[/csharp]

The interceptor will take the name of the interface defined method that is being acted on, make the first letter lowercase, and pass in the arguments to the client proxy source reference that it contains. When you do a `Clients.All.foo()` signalR does the exact same thing inside at runtime, we're just moving this to be wrapped by the dynamic proxy.

If you want to act on a specific client, the type is slightly different but it also implements `IClientProxy`:

[csharp]  
private IJsMethods CurrentClient  
{  
 get  
 {  
 return (Clients.Client(Context.ConnectionId) as ConnectionIdProxy).AsStrongHub\<IJsMethods\>();  
 }  
}  
[/csharp]

## Conclusion

While this doesn't touch the client side of things, you can easily fix that problem. Imagine tagging the interface with an attribute and auto generating signalR javascript client side wireups. Now you can manage all your sends and receives in one place, have them be strongly typed, and set yourself up for robust and safe code generation of boring boilerplate!

Like mentioned above, a full working project is available on my [github](https://github.com/devshorts/StronglyTypedSignalr) and you can see a running example at [appharbour](http://strongsignalr.apphb.com/).

