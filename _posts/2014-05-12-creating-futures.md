---
layout: post
title: Creating futures
date: 2014-05-12 08:00:22.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- async
- c#
- design patterns
- futures
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561421346;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4296;}i:1;a:1:{s:2:"id";i:3435;}i:2;a:1:{s:2:"id";i:3565;}}}}

permalink: "/2014/05/12/creating-futures/"
---
Futures (and promises) are a fun and useful [design pattern](http://en.wikipedia.org/wiki/Futures_and_promises) in that they help encapsulate asynchronous work into composable objects. That and they help hide away the actual asynchronous execution implementation. It doesn't matter if the future is finally resolved on the threadpool, in a new thread, or in an event loop (like nodejs).

Asynchronous work wrapped in futures has garnered a lot of attention in the javascript world since they alleviate the heavy use of nested callbacks to return a final result. But futures aren't limited to just javascript, the C# async keyword is a form of a future, Java has a futures class, and a lot of other languages have the ability to use futures.

In order to to demystify the concept of Futures lets build own version. Futures aren't hard to implement, even when you have a language that doesn't have them built in (or if you are on the .NET micro without async or Tasks). All we need to do is encapsulate a lambda and create an API that lets us chain deferred futures together. Lets look at a final unit test to demonstrate what we're trying to accomplish.

[csharp]  
private void TestFutureImpl()  
{  
 int count = 0;

Func\<int\> action = () =\>  
 {  
 Console.WriteLine("Running " + count);  
 Thread.Sleep(TimeSpan.FromMilliseconds(count \* 1000));  
 Console.WriteLine("Resolving " + count);

count++;  
 return count;  
 };

var future = new NewThreadFuture\<int\>(action).Then(action).Then(action);

Console.WriteLine("All setup, nonblock but now wait");

Thread.Sleep(TimeSpan.FromSeconds(2));

Console.WriteLine("Requesting result");

var result = future.Resolve();

Assert.AreEqual(3, result);  
}  
[/csharp]

With an output of

[csharp]  
All setup, nonblock but now wait  
Running 0  
Resolving 0  
Running 1  
Resolving  
Running  
Requesting result  
Resolving 2  
[/csharp]

You can see the deferred action waits for a certain period of time, so it could take some time to complete. But, with our future we can encapsulate this work, compose two other futures (that will evaluate after the first is complete), and finally when we _ask_ for the result it will either block until its done, or immediately return the result if it was evaluated.

For my purposes, I wrote an eager evaluated futures class, this means that once the future is instantiated it immediately starts to try and execute. The upside to this is that the result is available sooner, the downside is that if nobody requests the result you did work you didn't have to.

Lets take a look at what's really going on. Here is the basic skeleton of the future. I wrap the passed in function in another function that is responsible for exception handling, as well as notifying whoever else is listening that the action completed (by setting a manual rest event mutex). The other function is responsible for either waiting for the mutex to complete, or returning the completed result. Subclasses can implement the `Execute` method which would just either run the wrapped method in a new thread, or run it in a thread pool. It honestly doesn't matter how its executed, it can even be run synchronously if you wanted to!

[csharp]  
public abstract class Future\<T\>  
{  
 private bool \_isComplete;

private ManualResetEvent \_mutex = new ManualResetEvent(false);

private Exception \_ex;

private readonly object \_lock = new object();

private T \_result;

public Future(Func\<T\> function)  
 {  
 Execute(Wrapped(function));  
 }

protected abstract void Execute(Action wrapped);

private Action Wrapped(Func\<T\> function)  
 {  
 return () =\>  
 {  
 try  
 {  
 \_result = function();

lock (\_lock)  
 {  
 \_isComplete = true;  
 }  
 }  
 catch (Exception ex)  
 {  
 \_ex = ex;  
 }  
 finally  
 {  
 \_mutex.Set();  
 }  
 };  
 }

public T Resolve()  
 {  
 lock (\_lock)  
 {  
 if (\_isComplete)  
 {  
 return \_result;  
 }  
 }

\_mutex.WaitOne();

if (\_ex != null)  
 {  
 throw \_ex;  
 }

return \_result;  
 }

// ....  
[/csharp]

Now lets look at how to compose futures. The idea is you have one future, and the next future won't run until the first is complete. You can either pass in the result of the first future, or just run another action (with no input). From the users perspective you get one future that represents the result of all the composed actions. All we need to do is to create a new lambda that first resolves the previous one (via the closure), then executes the next one, all wrapped in a new future!

[csharp]  
public abstract Future\<T\> Then(Func\<T\> next);

protected Func\<T\> ThenWithoutResult(Func\<T\> next)  
{  
 return () =\>  
 {  
 Resolve();

return next();  
 };  
}

protected Func\<Y\> ThenWithResult\<Y\>(Func\<T, Y\> next)  
{  
 return () =\>  
 {  
 var previousResult = Resolve();

return next(previousResult);  
 };  
}

public abstract Future\<Y\> Then\<Y\>(Func\<T, Y\> next);  
[/csharp]

Look at the implementation of one of the subclasses:

[csharp]  
public class NewThreadFuture\<T\> : Future\<T\>  
{  
 public NewThreadFuture(Func\<T\> function) : base(function)  
 {  
 }

protected override void Execute(Action wrapped)  
 {  
 var runner = new Thread(new ThreadStart(wrapped));

runner.Start();  
 }

public override Future\<T\> Then(Func\<T\> next)  
 {  
 return new NewThreadFuture\<T\>(ThenWithoutResult(next));  
 }

public override Future\<Y\> Then\<Y\>(Func\<T, Y\> next)  
 {  
 return new NewThreadFuture\<Y\>(ThenWithResult(next));  
 }  
}  
[/csharp]

Simple! Now we can create asynchronous actions (remember that asynchronous just means nonblocking, but they are IN ORDER), and represent the entire workflow with a single future object. From an implementors perspective we can now also control how we execute the actions, whether on the threadpool or on new threads (or we can add other mechanisms if we want). This is because the future base class handles all the resolving synchronization making sure everything happens in order (just non-blocking).

For full source, check out my [github](https://github.com/devshorts/Playground/tree/master/Futures/Futures).

