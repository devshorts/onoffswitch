---
layout: post
title: Threadpooling in netduino
date: 2013-06-17 08:00:39.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- netduino
- threading
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560398911;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4394;}i:1;a:1:{s:2:"id";i:1587;}i:2;a:1:{s:2:"id";i:365;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/06/17/threadpooling-in-netduino/"
---
Sometimes you want to do asynchronous work without holding up your current thread but the work that needs to be done doesn't really warrant [the cost](http://msdn.microsoft.com/en-us/magazine/cc163552.aspx#S4) of spinning up a new thread (though what the exact cost is on an embedded environment I'm not sure).

This where [threadpooling](http://en.wikipedia.org/wiki/Thread_pool_pattern) comes into play. A threadpool has a certain amount of pre-spun up threads that you can re-use for actions. You push actions onto the threadpool and when there is an available thread it'll run your action. While threadpools aren't free (you still incur [context switching](http://en.wikipedia.org/wiki/Context_switch) and the initial overhead of firing up a thread) you can limit your context switches and minimize thread start/cleanup time by reusing threads. Threadpooling is a handy feature and C# has built in support for it, but it's [not in the .net micro framework](http://netmf.codeplex.com/workitem/78) so I decided to write my own.

For a basic threadpool manager it's really pretty simple. First you start with no threads. When someone queues an action into the threadpool we spin up our first thread. The thread's main body is an infinite loop that waits on a shared mutex. When the mutex is pulsed, threads in the threadpool try and de-queue the queued action. Whoever gets the action then safely executes it. When the maximum number of threads is created we don't spin any more up and just re-use what exists.

It's important to note the `ManualResetEvent`. This means that the mutex will stay signaled (i.e. wait's will exit immediately) until the queue is empty. If we used an `AutoResetEvent` then if you queued up too much too fast the threadpool would miss events that were added while all the threads were running.

[csharp]  
public static class ThreadUtil  
{  
 #region Data

/// \<summary\>  
 /// Synchronizes thread queue actions  
 /// \</summary\>  
 private static readonly object lockObject = new object();

/// \<summary\>  
 /// List storing our available threadpool threads  
 /// \</summary\>  
 private static readonly ArrayList \_availableThreads = new ArrayList();

/// \<summary\>  
 /// Queue of actions for our threadpool  
 /// \</summary\>  
 private static readonly Queue \_threadActions = new Queue();

/// \<summary\>  
 /// Wait handle for us to synchronize de-queuing thread actions  
 /// \</summary\>  
 private static readonly ManualResetEvent \_threadSynch = new ManualResetEvent(false);

/// \<summary\>  
 /// Maximum size of our thread pool  
 /// \</summary\>  
 private const int MaxThreads = 3;

#endregion

#region Thread Start

/// \<summary\>  
 /// Starts a new thread with an action  
 /// \</summary\>  
 /// \<param name="start"\>\</param\>  
 public static void Start(ThreadStart start)  
 {  
 try  
 {  
 var t = new Thread(start);  
 t.Start();  
 }  
 catch (Exception ex)  
 {  
 Debug.Print(ex.ToString());  
 }  
 }

#endregion

#region ThreadPooling

/// \<summary\>  
 /// Queues an action into the threadpool  
 /// \</summary\>  
 /// \<param name="start"\>\</param\>  
 public static void SafeQueueWorkItem(ThreadStart start)  
 {  
 lock (lockObject)  
 {  
 \_threadActions.Enqueue(start);

// if we haven't spun all the threads up, create a new one  
 // and add it to our available threads

if (\_availableThreads.Count \< MaxThreads)  
 {  
 var t = new Thread(ActionConsumer);  
 \_availableThreads.Add(t);  
 t.Start();  
 }

// pulse all waiting threads  
 \_threadSynch.Set();  
 }  
 }

/// \<summary\>  
 /// Main body of a threadpool thread. Indefinitely wait until  
 /// an action is queued. When an action is de-queued safely execute it  
 /// \</summary\>  
 private static void ActionConsumer()  
 {  
 while (true)  
 {  
 // wait on action pulse  
 \_threadSynch.WaitOne();

ThreadStart action = null;

// try and de-queue an action  
 lock (lockObject)  
 {  
 if (\_threadActions.Count \> 0)  
 {  
 action = \_threadActions.Dequeue() as ThreadStart;  
 }  
 else  
 {  
 // the queue is empty and we are in a critical section  
 // safely reset the mutex so that everyone waits  
 // until the next action is queued

\_threadSynch.Reset();  
 }  
 }

// if we got an action execute it  
 if (action != null)  
 {  
 try  
 {  
 action();  
 }  
 catch (Exception ex)  
 {  
 Debug.Print("Unhandled error in thread pool: " + ex);  
 }  
 }  
 }  
 }

#endregion  
}  
[/csharp]

