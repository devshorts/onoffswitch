---
layout: post
title: Handle reconnections to signalR host
date: 2012-08-21 14:38:32.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- ".NET"
- c#
- Rx
- SignalR
- tasks
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '830832847'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561163778;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:289;}i:1;a:1:{s:2:"id";i:2365;}i:2;a:1:{s:2:"id";i:4091;}}}}

permalink: "/2012/08/21/handle-reconnections-to-signalr-host/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/handle-reconnections-to-signalr-host/)_

[SignalR](https://github.com/SignalR/SignalR/) does a great job of dealing with reconnecting to a host when either the client disconnects or the server disconnects. This is pretty handy since it handles all the intricacies of a persistent http connection for you. But what it doesn't deal with is the initial negotiation to a server. If that fails you are stuck retrying yourself. I wrote a simple reconnection function that leverages the scheduling functionality of [Rx](http://msdn.microsoft.com/en-us/data/gg577609.aspx) to continuously try to reconnect to the server.

For our SignalR usage (version 0.5.2), I'm using the exposed [Hub](https://github.com/SignalR/SignalR/wiki/QuickStart-Hubs) functionality, not the[persistent connections](https://github.com/SignalR/SignalR/wiki/QuickStart-Persistent-Connections) since I liked the encapsulation that Hub's gave us. In the following example, we have a local member variable called `Connection` which is a `HubConnection` type created with this code.

[csharp]  
HubConnection Connection = new HubConnection(Url);  
[/csharp]

HubConnection has a Start method that you use to initialize connections to the Url.&nbsp; `Connection.Start()` internally creates an asynchronous task that looks like this, after unwrapping the nicely packaged methods:

[csharp]  
private Task Negotiate(IClientTransport transport)  
{  
 var negotiateTcs = new TaskCompletionSource\<object\>();

transport.Negotiate(this).Then(negotiationResponse =\>  
 {  
 VerifyProtocolVersion(negotiationResponse.ProtocolVersion);

ConnectionId = negotiationResponse.ConnectionId;

var data = OnSending();  
 StartTransport(data).ContinueWith(negotiateTcs);  
 })  
 .ContinueWithNotComplete(negotiateTcs);

var tcs = new TaskCompletionSource\<object\>();  
 negotiateTcs.Task.ContinueWith(task =\>  
 {  
 try  
 {  
 // If there's any errors starting then Stop the connection  
 if (task.IsFaulted)  
 {  
 Stop();  
 tcs.SetException(task.Exception);  
 }  
 else if (task.IsCanceled)  
 {  
 Stop();  
 tcs.SetCanceled();  
 }  
 else  
 {  
 tcs.SetResult(null);  
 }  
 }  
 catch (Exception ex)  
 {  
 tcs.SetException(ex);  
 }  
 },  
 TaskContinuationOptions.ExecuteSynchronously);

return tcs.Task;  
}  
[/csharp]

The comment above the `IsFaulted` check says that if the server fails to connect, an exception is set and the transport is closed``. Since SignalR utilizes the task parallel library we can just call Start() again and get a new task.

Here is the snippet we use to continuously reconnect:

[csharp]  
/// \<summary\>  
/// Handles if the connection start task fails and retries every 5 seconds until  
/// it succeeds  
/// \</summary\>  
/// \<param name="startTask"\>\</param\>  
/// \<param name="connectionSucessAction"\>\</param\>  
private void HandleConnectionStart(Task startTask)  
{  
 startTask.ContinueWith(task =\>  
 {  
 try  
 {  
 if (task.IsFaulted)  
 {  
 // make sure to observe the exception or we can get an aggregate exception  
 foreach (var e in task.Exception.Flatten().InnerExceptions)  
 {  
 Log.WarnOnce(this, "Observed exception trying to handle connection start: " + e.Message);  
 }

Log.WarnOnce(this, "Unable to connect to url {0}, retrying every 5 seconds", Url);  
 RetryConnectionStart();  
 }  
 else  
 {  
 // do success actions  
 }  
 }  
 catch(Exception ex)  
 {  
 Log.ErrorOnce(this, "Error handling connection start, retrying", ex);  
 RetryConnectionStartRescheduler();  
 }  
 });  
}

private void RetryConnectionStartRescheduler()  
{  
 ThreadUtil.ScheduleToThreadPool(TimeSpan.FromSeconds(5),  
 () =\>  
 {  
 try  
 {  
 HandleConnectionStart(Connection.Start());  
 }  
 catch(Exception ex)  
 {  
 Log.ErrorOnce(this, "Error retrying connection start, retrying", ex);  
 RetryConnectionStartRescheduler();  
 }  
 });

}  
[/csharp]

`ThreadUtil.ScheduleToThreadPool` is a wrapper we have on top of the Rx framework's threadpool scheduler. Internally it looks like this

[csharp]  
public static void ScheduleToThreadPool(TimeSpan executeTime, Action action)  
{  
 Scheduler.ThreadPool.Schedule(DateTime.Now.Add(executeTime), action);  
}  
[/csharp]

It's important to note that you have to touch the exception object of a faulted task or use the exception Handle method in order to avoid an [UnobservedTaskExceptions](http://msdn.microsoft.com/en-us/library/dd997415.aspx). Those happen to unobserved exceptions which are then rethrown on the finalizer thread.

In conclusion, by leveraging tasks and a couple simple scheduling utilities, we can cleanly and asynchronously schedule a new task to connect periodically. When we finally connect we can continue with our initialization logic. At this point the remaining signalR reconnection logic is handled by the hub.

