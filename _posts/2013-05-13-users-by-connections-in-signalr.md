---
layout: post
title: Users by connections in SignalR
date: 2013-05-13 08:00:09.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- ".NET"
- SignalR
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561793453;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3392;}i:1;a:1:{s:2:"id";i:4116;}i:2;a:1:{s:2:"id";i:155;}}}}

permalink: "/2013/05/13/users-by-connections-in-signalr/"
---
SignalR gives you events when users connect, disconnect, and reconnect, however the only identifying piece of information you have at this point is their connection ID. Unfortunately it's not very practical to identify all your connected users strictly off their connectionIDs - usually you have some other identifier in your application (userID, email, etc).

If you are using ASP.NET MVC3, you can access this information from the hub context via [`Context.User`](http://stackoverflow.com/questions/7289724/how-can-i-access-the-logged-in-user-from-outside-of-a-controller), but if you aren't using mvc3 (like a .net to .net client) a good workflow is to have your client identify themselves on connect. They can call a known `Register` method on the hub and give you the identifying information of who they are.

At this point you have your unique identifier along with their connectionID, but you have to manage all their disconnections, reconnections, and multiple connections of the same client yourself. This post will go through an easy way to manage this client information yourself.

## The Hub

I like to keep the hub code simple and use it only as an incoming facade. I hand off every request to an internal singleton that I can then leverage and put all the business logic in. This keeps the hub clean.

```csharp
  
[HubName("hub")]  
public class ServerHub : Hub, IConnected, IDisconnect  
{  
 private HubProcessor Instance  
 {  
 get { return HubProcessor.Instance; }  
 }

public ServerHub()  
 {  
 Instance.Initialize();  
 }

public void Register(HubClientPayload payload)  
 {  
 Instance.Register(payload, Context.ConnectionId);  
 }

public Task Connect()  
 {  
 return Instance.Connect(Context.ConnectionId);  
 }

public Task Reconnect(IEnumerable\<string\> groups)  
 {  
 return Instance.Reconnect(Context.ConnectionId);  
 }

public Task Disconnect()  
 {  
 return Instance.Disconnect(Context.ConnectionId);  
 }  
}  

```

## Hub Processor

I've put all the business logic of the hub processing into a hub processor singleton that all instances of the hub can access. When a hub is initialized it calls an Initialize function on the singleton which is only used to force lazy creation on the singleton:

```csharp
  
public class HubProcessor : HubProcessorBase\<AusUpdaterHub\>  
{  
 #region Data

private static object \_lock = new object();

#endregion

#region Singleton and Constructor

private static readonly Lazy\<HubProcessor\> \_instance = new Lazy\<HubProcessor\>();

public static HubProcessor Instance  
 {  
 get { return \_instance.Value; }  
 }

public void Initialize()  
 {  
 // force creation of lazy constructor  
 }

#endregion

// ... business logic handed off from the hub  
}  

```

The HubProcessor inherits from a base class which only exposes a helper method to get the context for a current hub. This is so we can re-use the base class elsewhere, or if we want to create our own [hub context wrappers](https://github.com/blinemedical/SignalRToAs3) we can do that in the base class without affecting how the hub treats a client.

```csharp
  
public class HubProcessorBase\<T\> : IDisposable where T: Hub  
{  
 protected IHubContext Context  
 {  
 get  
 {  
 return GlobalHost.ConnectionManager.GetHubContext\<T\>();  
 }  
 }

protected override void Dispose()  
 {  
 // for inheritance  
 }  
}  

```

## Connect

When a client connects, we don't know anything about them other than their connection ID. If we're not using MVC3 then we'll need the client to tell us who they are and give us some meaningful information. The expectation is that they will register themselves when they successfully connect (which they can know about client side).

```csharp
  
/// \<summary\>  
/// Called by a client when they connect and register  
/// \</summary\>  
/// \<param name="payload"\>\</param\>  
/// \<param name="connectionId"\>\</param\>  
public void Register(HubClientPayload payload, string connectionId)  
{  
 try  
 {  
 lock (\_lock)  
 {  
 List\<String\> connections;  
 if (\_registeredClients.TryGetValue(payload.UniqueID, out connections))  
 {  
 if (!connections.Any(connection =\> connectionID == connection))  
 {  
 connections.Add(connectionId);  
 }  
 }  
 else  
 {  
 \_registeredClients[payload.UniqueID] = new List\<string\> { connectionId };  
 }  
 }  
 }  
 catch(Exception ex)  
 {  
 Log.Error(this, "Error registering on hub", ex);  
 }  
}  

```

When a client registers on the hub, the hub passes the input argument (the client payload, which contains unique identifying information) as well as the connectionID to the hub processor. Now we have a thread safe dictionary that tracks the users unique identifier along with all associated connectionIDs. This way if the same user is open in multiple tabs, or across multiple .net clients, we can have a central list of connectionIDs to act on.

## Disconnect

When a client disconnects we'll execute the disconnect function on the singleton which will remove the connection from the connected client list

```csharp
  
/// \<summary\>  
/// Invoked by SignalR when a disconnection is detected  
/// \</summary\>  
/// \<param name="connectionId"\>\</param\>  
public Task Disconnect(string connectionId)  
{  
 try  
 {  
 lock (\_lock)  
 {  
 var connections = \_registeredClients.Where(c =\> c.Value.Any(connection =\> connection == connectionId)).FirstOrDefault();

// if we are tracking a client with this connection  
 // remove it  
 if (!CollectionUtil.IsNullOrEmpty(connections.Value))  
 {  
 connections.Value.Remove(connectionId);

// if there are no connections for the client, remove the client from the tracking dictionary  
 if (CollectionUtil.IsNullOrEmpty(connections.Value))  
 {  
 \_registeredClients.Remove(connections.Key);  
 }  
 }  
 }  
 }  
 catch(Exception ex)  
 {  
 Log.Error(this, "Error on disconnect in hub", ex);  
 }

return null;  
}  

```

## Reconnections

Now, what happens if our server goes down but clients are still up? When they come online they'll do a reconnect, not an initial connect. When clients reconnect we should just invoke back to them to re-register themselves. This way we can quickly rebuild our tracker dictionary of who is out there. You might want to persist the dictionary and validate who is still connected by the reconnection message, but what happens if a client is slow to reconnect? At what point do we invalidate disconnected clients? I think it's safer to have everyone re-register. You can obviously throttle this by having the re-registration synchronized or batched off if you have a huge number of connected clients.

```csharp
  
/// \<summary\>  
/// Invoked by SignalR when a client reconnects to the server  
/// \</summary\>  
/// \<param name="connectionId"\>\</param\>  
public Task Reconnect(string connectionId)  
{  
 try  
 {  
 Context.Clients[connectionId].reRegister();  
 }  
 catch(Exception ex)  
 {  
 Log.Error(this, "Error re-connecting on hub", ex);  
 }

return null;  
}  

```

## Invoking

At this point we have a dictionary keyed off our internal user unique identifier. To do any work all we have to do is lock the registeredClients dictionary, get the list of connectionID's associated to who we want, and execute them on the context. For example:

```csharp
  
private void SendTextToUser(string uniqueID, string text)  
{  
 DispatchToClient(connection =\> connection.sendText(text), uniqueID);  
}

/// \<summary\>  
/// Execute lambda for each connection associated to a client's unique ID  
/// \</summary\>  
/// \<param name="action"\>\</param\>  
/// \<param name="uniqueID"\>\</param\>  
private void DispatchToClient(Action\<dynamic\> action, string uniqueID)  
{  
 foreach (dynamic connection in GetConnections(uniqueID))  
 {  
 action(connection);  
 }  
}

private List\<dynamic\> GetConnections(string uniqueID)  
{  
 var connections = new List\<dynamic\>();  
 lock (\_lock)  
 {  
 connections = (from client in \_registeredClients  
 let clientID = client.Key  
 let clientConnections = client.Value  
 where clientID == uniqueID  
 from connection in clientConnections  
 select Context.Clients[connection]).ToList();  
 }  
 return connections;  
}  

```

## Conclusion

The nice thing about having things set up this way is that as long as there are some connections associated to a user we know that that user is active.

Disconnects, depending on what long polling transport mechanism SignalR chooses for the client, can sometimes come after a short timeout. This means that a user can disconnect, and then create a new connection. Maybe they closed the browser window, then opened it back up again. In this scenario for a short time we'll have two connections, but only one is an actual valid connection for the client. We won't know that the first one disconnected until the timeout expires. But, that's ok, since the dictionary guarantees you can get to at least one connection for the active client. After a short period of time the disconnect message will happen and we can do a cleanup.

While SignalR does great job of letting broadcast info based on connectionID, we sometimes want to have our own collections pairing connectionIDs to other client identifying information. With some simple client to server invocations and a threadsafe dictionary we can keep track of all the relevant information stored in an easy to use fashion.

