---
layout: post
title: Thread Synchronization With Aspects
date: 2013-02-11 14:52:05.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- aspects
- c#
- postsharp
- synchronization
- threading
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  dsq_thread_id: '1070974672'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558136896;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4394;}i:1;a:1:{s:2:"id";i:738;}i:2;a:1:{s:2:"id";i:383;}}}}

permalink: "/2013/02/11/thread-synchronization-with-aspects/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/thread-synchronization-with-aspects/)_

[Aspect-oriented programming](http://onjava.com/pub/a/onjava/2004/01/14/aop.html) is an interesting way to decouple common method level logic into localized methods that can be applied on build. For C#, [PostSharp](http://www.sharpcrafters.com/) is a great tool that does the heavy lifting of the MSIL rewrites to inject itself in and around your methods based on method tagging with attributes. PostSharp's offerings are split up into [free aspects and pro aspects](http://www.sharpcrafters.com/postsharp/features) so it makes diving into aspect-oriented programming easy since you can get a lot done with their free offerings.

One of their free aspects, the [method interception aspect](http://doc.sharpcrafters.com/postsharp-2.1/##PostSharp-2.1.chm/html/T_PostSharp_Aspects_MethodInterceptionAspect.htm), lets you control how a method gets invoked. Using this capability, my general idea was to expose some sort of lock and wrap the method invocation automatically in lock statement using a shared object. This way, we can manage thread synchronization using aspects.

Managing thread synchronization with aspects isn't a new idea: the PostSharp site already has [an example](http://www.sharpcrafters.com/solutions/locking) of thread synchronization. However, they are using a pro feature aspect that allows them to auto-implement a new interface for tagged classes. For the purposes of my example, we can do the same thing without using the pro feature and simultaneously add a little extra functionality.

There are two things I wanted to accomplish. One was to simplify local method locking (basically what the PostSharp example solves), and the second was to facilitate locking of objects across multiple files and namespace boundaries. You can imagine a situation where you have two or more singletons who work on a shared resource. These objects need some sort of shared lock reference to synchronize on, which means you need to expose the synchronized object between all the classes. Not only does this tie classes together, but it can also get messy and error-prone as your application grows.

First, I've defined an interface that exposes a basic lock. Implementing the interface is optional as you'll see later.

```csharp
  
public interface IAspectLock  
{  
 object Lock { get; }  
}  

```

Next we have the actual aspect we'll be tagging methods with.

```csharp
  
[Serializable]  
public class Synchronize : MethodInterceptionAspect  
{  
 private static readonly object FlyweightLock = new object();

private static readonly Dictionary\<string, object\> LocksByName = new Dictionary\<string, object\>();

private String LockName { get; set; }

/// \<summary\>  
 /// Constructor when using a shared lock by name  
 /// \</summary\>  
 /// \<param name="lockName"\>\</param\>  
 public Synchronize(String lockName)  
 {  
 LockName = lockName;  
 }

/// \<summary\>  
 /// Constructor for when an object implements IAspectLock  
 /// \</summary\>  
 public Synchronize()  
 {

}

public override void OnInvoke(MethodInterceptionArgs args)  
 {  
 object locker;

if (String.IsNullOrEmpty(LockName))  
 {  
 var aspectLockObject = args.Instance as IAspectLock;

if (aspectLockObject != null)  
 {  
 locker = aspectLockObject.Lock;  
 }  
 else  
 {  
 throw new Exception(String.Format("Method {0} didn't define a lock name nor implement IAspectLock", args.Method.Name));  
 }  
 }  
 else  
 {  
 lock (FlyweightLock)  
 {  
 if (!LocksByName.TryGetValue(LockName, out locker))  
 {  
 locker = new object();  
 LocksByName[LockName] = locker;  
 }  
 }  
 }

lock (locker)  
 {  
 args.Proceed();  
 }  
 }  
}  

```

The attribute can either take a string representing the name of the global lock we want to use, or, if none is provided, we can test to see if the instance implements our special interface and use its lock. When an object implements `IAspectLock` the code path is simple: get the lock from the object and use it on the method.

The second code path, when you use global lock name, lets you lock across the entire application without having to tie classes together, keeping things clean and decoupled.

For the scenario where a global lock name was defined, I used a static dictionary to keep track of the locks and corresponding reference objects to lock on based on name. This way I can maximize throughput by using a flyweight container: lock first on the dictionary just to get the lock I want, then lock on the value retrieved. The locking of the dictionary will always be fast and shouldn't be contended for that often. Uncontested locks are tested for using [spinlock semantics](http://tech.blinemedical.com/inter-process-locking/) so they are usually extremely quick. Once you have the actual lock you want to use for this function, you can call `args.Proceed()` which will actually invoke the tagged method.

Just to be sure this all works, I wrote a unit test to make sure the attribute worked as expected. The test spawns 10,000 threads which will each loop 100,000 times and increment the `_syncTest` integer. The idea is to introduce a race condition. Given enough threads and enough work, some of those threads won't get the updated value of the integer and won't actually increment it. For example, at some point both threads may think `_syncTest` is 134, and both will increment to 135. If it was synchronized, the value, after two increments, should be 136. Since race conditions are timing-dependent we want to make the unit test stressful to try and maximize the probability that this would happen. Theoretically, we could run this test and never get the race condition we're expecting, since that's by definition a race condition (non-deterministic results). However, on my machine, I was able to consistently reproduce the expected failure conditions.

```csharp
  
private int \_syncTest = 0;  
private const int ThreadCount = 10000;  
private const int IterationCount = 100000;

[Test]  
public void TestSynchro()  
{  
 var threads = new List\<Thread\>();  
 for (int i = 0; i \< ThreadCount; i++)  
 {  
 threads.Add(ThreadUtil.Start("SyncTester" + i, SynchroMethod));  
 }

threads.ForEach(t=\>t.Join());

Assert.True(\_syncTest == ThreadCount \* IterationCount,  
 String.Format("Expected synchronized value to be {0} but was {1}", ThreadCount \* IterationCount, \_syncTest));  
}

[GlobalSynchronize("SynchroMethodTest")]  
private void SynchroMethod()  
{  
 for (int i = 0; i \< IterationCount; i++)  
 {  
 \_syncTest++;  
 }  
}  

```

When the method doesn't have the attribute we get an NUnit failure like

```csharp
  
 Expected synchornized value to be 1000000000 but was 630198141  
 Expected: True  
 But was: False

at NUnit.Framework.Assert.That(Object&nbsp;actual,&nbsp;IResolveConstraint&nbsp;expression,&nbsp;String&nbsp;message,&nbsp;Object[]&nbsp;args)  
 at NUnit.Framework.Assert.True(Boolean&nbsp;condition,&nbsp;String&nbsp;message)  
 at AspectTests.TestSynchro() in AspectTests.cs: line 35  

```

Showing the race condition that we expected did happen (the value will change each time). When we have the method synchronized, our test passes.

