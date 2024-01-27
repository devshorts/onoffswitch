---
layout: post
title: Single producer many consumer
date: 2014-02-26 22:21:12.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- async
- queue
- Rx
- topic
meta:
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561845469;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:532;}i:1;a:1:{s:2:"id";i:2985;}i:2;a:1:{s:2:"id";i:7777;}}}}

permalink: "/2014/02/26/single-producer-consumer/"
---
When I'm bored, I like to roll my own versions of things that already exist. That's not to say I use them in production, but I find that they are great learning tools. If you read the blog regularly you probably have realized I do this A LOT. Anyways, today is no different. I was thinking about single producer, multiple consumer functions, like an [SNS](http://aws.amazon.com/sns/) Topic, but for your local machine. In reality, the best way to do this would be to publish your event through an Rx stream and consume it with multiple subscribers, but that's no fun. I want to roll my own!

BlockingCollection in .NET supports thread safe multiple consumers, but only 1 item will ever get dequeued from your collection. That means that if you have multiple threads waiting on a consuming enumerable, only one of them will get a result (not all of them). That's not that good if you want to have copies of your item dispatched to multiple subscribers. But, if that is what you want, check out this other [post of mine](http://onoffswitch.net/async-producerconsumer-the-easy-way/).

What I need is a blocking consumer, a way to publish items, a threadsafe way to add and remove subscriptions, and a way to concurrently dispatch dequeued items to subscribed consumers.

First let me show a subscriber instance. This is like a public token that the consumer of the topic will get when they subscribe. All it has is a registered `OnNext` action and a way to unsubscribe itself from whatever its subscribed to.

```csharp
  
public class Subscriber\<T\>{  
 private Action\<Subscriber\<T\>\> UnSubscribeAction { get; set; }

public Action\<T\> OnNext{ get; private set; }

public void UnSubscribe(){  
 UnSubscribeAction (this);  
 }

public Subscriber(Action\<Subscriber\<T\>\> unsubscribe, Action\<T\> onNext){  
 UnSubscribeAction = unsubscribe;

OnNext = onNext;  
 }  
}  

```

And now the actual single producer many consumer (SPMC) implementation. It's responsible for handling the listening on the consuming enumerable, the dispatching into the blocking collection, as well as parallelizing the re-distribution of the consumers. It's pretty simple!

```csharp
  
public class SPMC\<T\> : IDisposable  
{  
 public SPMC (int boundedSize = int.MaxValue)  
 {  
 \_blockingCollection = new BlockingCollection\<T\> (boundedSize);  
 }

private Object \_lock = new object();

private List\<Subscriber\<T\>\> \_consumers = new List\<Subscriber\<T\>\>();

private BlockingCollection\<T\> \_blockingCollection;

public Subscriber\<T\> Subscribe(Action\<T\> onNext){  
 lock (\_lock) {

Action\<Subscriber\<T\>\> removalAction = instance =\> {  
 lock (\_lock) {  
 \_consumers.Remove (instance);  
 }  
 };

var subscriber = new Subscriber\<T\> (removalAction, onNext);

\_consumers.Add (subscriber);

return subscriber;  
 }  
 }

public void Start(){  
 new Thread (() =\> {  
 foreach (var item in \_blockingCollection.GetConsumingEnumerable()) {  
 lock (\_lock) {  
 Parallel.ForEach (\_consumers, consumer =\> consumer.OnNext (item));  
 }  
 }  
 }).Start();  
 }

public void Stop(){  
 \_blockingCollection.CompleteAdding ();  
 }

public void Publish(T item){  
 \_blockingCollection.Add (item);  
 }

#region IDisposable implementation

public void Dispose ()  
 {  
 Stop ();  
 }

#endregion  
}  

```

And of course, a unit test to demonstrate its usage

```csharp
  
[Test]  
public void TestCase ()  
{  
 var subscriber1Collect = new List\<string\> ();  
 var subscriber2Collect = new List\<string\> ();  
 var subscriber3Collect = new List\<string\> ();

var spmc = new SPMC\<String\> ();

var subscriber1 = spmc.Subscribe(subscriber1Collect.Add);

spmc.Subscribe(subscriber2Collect.Add);

spmc.Start ();

var t = new Thread (() =\> {  
 while (true) {  
 Thread.Sleep (TimeSpan.FromMilliseconds (1000));

spmc.Publish (DateTime.Now.ToString ());  
 }  
 });

t.IsBackground = true;

t.Start ();

Thread.Sleep (TimeSpan.FromSeconds (5));

subscriber1.UnSubscribe ();

spmc.Subscribe(subscriber3Collect.Add);

Thread.Sleep (TimeSpan.FromSeconds (5));

Assert.GreaterOrEqual (subscriber3Collect.Count, 4);  
 Assert.GreaterOrEqual (subscriber2Collect.Count, 9);  
 Assert.GreaterOrEqual (subscriber1Collect.Count, 4);  
}  

```

The asserts are greater than or equal just to give a 1 second wiggle room for the time dispatch variance.

