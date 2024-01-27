---
layout: post
title: When to abort a thread
date: 2013-03-18 08:00:06.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- Best Practices
- c#
- threads
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554802472;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2985;}i:1;a:1:{s:2:"id";i:532;}i:2;a:1:{s:2:"id";i:4764;}}}}

permalink: "/2013/03/18/when-to-abort-a-thread/"
---
When is it OK to abort a thread is a question that comes up every so often. Usually everyone jumps on the bandwagon that you should never ever do a thread abort, but I don't agree. Certainly there are times when it's valid and if you understand what you are doing then it's ok to use.

The reasoning behind never using thread abort is because calling abort on a thread issues an asynchronous exception, meaning that exceptions could happen where you think there never should be exceptions such as dispose methods or finally blocks. [This post](http://www.interact-sw.co.uk/iangblog/2004/11/12/cancellation) describes what happens with thread abort and I found it to be a good read.

But, I still don't think you should _never_ use thread abort. The big issue is what if you don't have access to the code that is running in the thread? If a 3rd party library is blocking your app or is doing something uncontrollable and you need to end it you don't have a lot of options to gracefully exit. You may not be able to recompile the code and sprinkle in some cancellation tokens, or maybe you aren't comfortable with doing that since the library code is large and you're not familiar with its internals.

If the problem is that your application won't shut down because of a runaway thread, people sometimes suggest to run the work in a background thread, but I still think that's a bad idea. Background threads [are the same](http://msdn.microsoft.com/en-us/library/system.threading.thread.isbackground.aspx) as foreground threads, except they won't keep your application from exiting if they are still running. So background threads can still block indefinitely or do whatever they want throughout the course of your application.

When I'm faced with this kind of a prospect, and I certainly don't come across this often, I like to make sure to sandbox that code in a thread that I can control. Below is a method I use that helps me isolate problem areas when I come into these kinds of scenarios. This is honestly a last resort. It's always better to know why something isn't working as expected. Unfortunately, somtimes in the real world you can't always devote the time, or even figure it out, so you have to work around things.

Here is `RunWithTimeout` and I hope the block comment is pretty clear.

```csharp
  
/// \<summary\>  
/// Helper function to wrap actions within another thread and test to see how long its run.  
/// Only allows the Action() to run within the alloted time or it'll abort the wrapped thread.  
///  
/// BAD PRACTICE TO USE THIS IF YOU CONTROL THE CODE!!!  
///  
/// This function is mostly for wrapping 3rd party components that are blocking and have the potential to be  
/// "runaway". For example, if we're zipping a stream and the input stream continues to grow and the zip library  
/// doesn't allow us to exit until the next entry. At which point we'll never be able to cancel. This is a good example  
/// of when to use this function  
/// \</summary\>  
/// \<param name="action"\>the function to wrap in a timeout\</param\>  
/// \<param name="timeout"\>how long we'll let the function run\</param\>  
/// \<param name="description"\>the name of the timeout thread\</param\>  
/// \<param name="checkTimeMs"\>how often the internal thread should check to see if the timeout has occured\</param\>  
/// \<returns\>Returns true if the thread executed within the allotted timeout, or false if an exception or timeout occurred\</returns\>  
public static bool RunWithTimeout(Action action, TimeSpan timeout, string description, int checkTimeMs = 250)  
{  
 try  
 {  
 var startTime = DateTime.Now;  
 var thread = ThreadUtil.Start(description + "-TimeoutThread", () =\> action());  
 while (true)  
 {  
 var runTime = DateTime.Now - startTime;  
 if (runTime \>= timeout && thread.IsAlive)  
 {  
 try  
 {  
 thread.Abort();  
 }  
 catch (Exception ex)  
 {  
 Log.Error(typeof (TimeoutUtil),  
 String.Format("Unable to abort runaway thread that has excceded timeout {0}",  
 timeout), ex);  
 }

return false;  
 }

if (!thread.IsAlive)  
 {  
 return true;  
 }

Thread.Sleep(TimeSpan.FromMilliseconds(checkTimeMs));  
 }  
 }  
 catch(Exception ex)  
 {  
 Log.Error(typeof(TimeoutUtil), "Unknown error executing action with timeout", ex);  
 return false;  
 }  
}  

```

The basic idea here is to spin up two threads. One that does the actual work, and another to wait for a period of time, check if the first thread is done, and if not, forcibly abort it.

And here is an NUnit test to demonstrate it. We're going to set a timeout of one second, but have our sandboxes function wait for two seconds. This means the sandbox function should be aborted since it's taking too long. We can then test to see how long we blocked for (it should be around one second) and validate that `RunWithTimeout` returned `false` indicating that it uncleanly exited and was aborted.

```csharp
  
[Test]  
public void TestTimeout()  
{  
 var start = DateTime.Now;

var didTimeOut =  
 TimeoutUtil.RunWithTimeout(  
 () =\> WaitAction(TimeSpan.FromSeconds(2)),  
 TimeSpan.FromSeconds(1),  
 "1secondsWithTimeout");

var runTime = DateTime.Now - start;

Console.WriteLine("Action took " + runTime.TotalSeconds);

Assert.False(didTimeOut);  
}

private static void WaitAction(TimeSpan wait)  
{  
 Thread.Sleep(wait);  
}  

```

