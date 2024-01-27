---
layout: post
title: Async producer/consumer the easy way
date: 2012-11-23 10:56:35.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- c#
- design patterns
meta:
  _syntaxhighlighter_encoded: '1'
  _wp_old_slug: asynch-producerconsumer-the-easy-way
  _edit_last: '1'
  dsq_thread_id: '940776371'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561850503;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4435;}i:1;a:1:{s:2:"id";i:4394;}i:2;a:1:{s:2:"id";i:2447;}}}}

permalink: "/2012/11/23/async-producerconsumer-the-easy-way/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/async-producerconsumer-the-easy-way/)_

In .net 4, a new class called [`BlockingCollection`](http://msdn.microsoft.com/en-us/library/dd267312.aspx) was introduced, which let you have a [threadsafe](http://en.wikipedia.org/wiki/Thread_safety) [producer/consumer](http://en.wikipedia.org/wiki/Producer-consumer_problem) queue. Anyone consuming a `BlockingCollection` blocks automatically until new items are added. This lets you easily add items to the collection in one thread and use another synchronized thread to consume items. This class is great since before this existed, you had to do all this work with mutexes and it was a lot of extra work (and more error prone). In general, a good time to use a decoupled producer consumer pattern is when you have a slow consuming function and a producer thread that is time sensitive.

Even though `BlockingCollection` effectively synchronizes your producer/consumer, you still have to create [boilerplate](http://en.wikipedia.org/wiki/Boilerplate_code) to manage the producer thread and the consumer thread. Also if you wanted to add extra exception handling or a [cancellation token,](http://msdn.microsoft.com/en-us/library/dd997364.aspx) you'd have to add all that yourself too. I wrapped this all up in a `BlockingCollectionWrapper` class that handles all this for you.

# An example

Here is an example where the consumer takes one second each time it consumes an item.

```csharp
  
private readonly ManualResetEvent \_testMutex = new ManualResetEvent(false);

[Test]  
public void TestCollection()  
{  
 // create the wrapper  
 var asyncCollection = new BlockingCollectionWrapper\<string\>();

asyncCollection.FinishedEvent += FinishedEventHandler;

// make sure we dispose of it. this will stop the internal thread  
 using (asyncCollection)  
 {  
 // register a consuming action  
 asyncCollection.QueueConsumingAction = (producedItem) =\>  
 {  
 Thread.Sleep(TimeSpan.FromSeconds(1));  
 Console.WriteLine(DateTime.Now + ": Consuming item: " + producedItem);  
 };

// start consuming  
 asyncCollection.Start();

// start producing  
 for (int i = 0; i \< 10; i++)  
 {  
 Console.WriteLine(DateTime.Now + ": Produced item " + i);  
 asyncCollection.AddItem(i.ToString());  
 }  
 }

// wait for the finished handler to pulse this  
 \_testMutex.WaitOne();

Assert.True(asyncCollection.Finished);  
}

private void FinishedEventHandler(object sender, BlockingCollectionEventArgs e)  
{  
 \_testMutex.Set();  
}  

```

This prints out

```csharp
  
9/17/2012 6:22:43 PM: Produced item 0  
9/17/2012 6:22:43 PM: Produced item 1  
9/17/2012 6:22:43 PM: Produced item 2  
9/17/2012 6:22:43 PM: Produced item 3  
9/17/2012 6:22:43 PM: Produced item 4  
9/17/2012 6:22:43 PM: Produced item 5  
9/17/2012 6:22:43 PM: Produced item 6  
9/17/2012 6:22:43 PM: Produced item 7  
9/17/2012 6:22:43 PM: Produced item 8  
9/17/2012 6:22:43 PM: Produced item 9  
9/17/2012 6:22:44 PM: Consuming item: 0  
9/17/2012 6:22:45 PM: Consuming item: 1  
9/17/2012 6:22:46 PM: Consuming item: 2  
9/17/2012 6:22:47 PM: Consuming item: 3  
9/17/2012 6:22:48 PM: Consuming item: 4  
9/17/2012 6:22:49 PM: Consuming item: 5  
9/17/2012 6:22:50 PM: Consuming item: 6  
9/17/2012 6:22:51 PM: Consuming item: 7  
9/17/2012 6:22:52 PM: Consuming item: 8  
9/17/2012 6:22:53 PM: Consuming item: 9  

```

First, I created the blocking collection wrapper and made sure to put it in a `using` block since it's disposable (the thread waiting on the blocking collection will need to be cleaned up). Then I registered a function to be executed each time an item is consumed. Calling `Start()` begins consuming. Once I'm done - even after the using block disposes of the wrapper - the separate consumer thread could still be running (processing whatever is left), but it is no longer blocking on additions and will complete consuming any pending items.

# The wrapper

When you call `.Start()` we start our independent consumer thread.

```csharp
  
/// \<summary\>  
/// Start the consumer  
/// \</summary\>  
public void Start()  
{  
 \_cancellationTokenSource = new CancellationTokenSource();  
 \_thread = new Thread(QueueConsumer) {Name = "BlockingConsumer"};  
 \_thread.Start();  
}  

```

This is the queue consumer that runs in the separate thread that executes the registered consumer action. The consuming action is locked to make changing the consuming action threadsafe.

```csharp
  
/// \<summary\>  
/// The actual consumer queue that runs in a seperate thread  
/// \</summary\>  
private void QueueConsumer()  
{  
 try  
 {  
 // Block on \_queue.GetConsumerEnumerable  
 // When an item is added to the \_queue it will unblock and let us consume  
 foreach (var item in \_queue.GetConsumingEnumerable(\_cancellationTokenSource.Token))  
 {  
 // get a synchronized snapshot of the action  
 Action\<T\> consumerAction = QueueConsumingAction;

// execute our registered consuming action  
 if (consumerAction != null)  
 {  
 consumerAction(item);  
 }  
 }

// dispose of the token source  
 if (\_cancellationTokenSource != null)  
 {  
 \_cancellationTokenSource.Dispose();  
 }

//Log.Debug(this, "Done with queue consumer");

Finished = true;

if (FinishedEvent != null)  
 {  
 FinishedEvent(this, new BlockingCollectionEventArgs());  
 }  
 }  
 catch(OperationCanceledException)  
 {  
 //Log.Debug(this, "Blocking collection\<{0}\> cancelled", typeof(T));  
 }  
 catch (Exception ex)  
 {  
 //Log.Error(this, ex, "Error consuming from queue of type {0}", typeof(T));  
 }  
}  

```

And when the wrapper is disposed, we set `CompleteAdding` on the blocking collection which tells the collection to stop waiting for new additions and finish out whatever is left in the queue.

```csharp
  
protected void Dispose(bool disposing)  
{  
 if(disposing)  
 {  
 if (\_queue !=null && !\_queue.IsAddingCompleted)  
 {  
 // mark the queue as complete  
 // the BlockingConsumer thread will now  
 // just process the remaining items  
 \_queue.CompleteAdding();  
 }  
 }  
}

public void Dispose()  
{  
 Dispose(true);  
}  

```

The remaining properties and functions on the wrapper let you

- Force abort the consumer thread
- Register a Finished event handler; disposing of the wrapper doesn't mean that no more work is being done. It means that you are no longer adding items and the queue is effectively "closed". Depending on your consumer function though, this could take some time to complete. This is why it's good to hook into the finished event so you can be sure that all your processing is complete.
- Manually mark the queue as AddedComplete (so the thread stops blocking)
- Manually cancel the queue
- Check if the queue is ended by looking at the `Finished` property

So to reiterate, the basic idea here is

- Create a separate thread that has appropriate exception handling to be blocked while consuming the queued items
- Handle cancellation gracefully
- Be able to properly end our spawned thread so we don't have anything leftover

It should be noted that even though this wrapper is built for a single consumer/single producer design, since we are leveraging `GetConsumingEnumerable` we could modify the wrapper to allow for [multiple threads acting as consumers](http://stackoverflow.com/questions/7528173/multiple-consumers-and-querying-a-c-sharp-blockingcollection) on the same enumerable. This could give us a single producer/multiple synchronized consumer pattern where only one consumer thread gets the particular item but multiple consumer threads exist and can do work.

Full source and tests provided at our [github](https://github.com/blinemedical/BlockingCollectionWrapper).

