---
layout: post
title: RxJava Observables and Akka actors
date: 2014-03-07 23:41:36.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- actors
- akka
- functional reactive
- java
- Rx
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561910971;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4627;}i:1;a:1:{s:2:"id";i:4593;}i:2;a:1:{s:2:"id";i:4629;}}}}

permalink: "/2014/03/07/rxjava-observables-akka-actors/"
---
I was playing with both [akka](http://akka.io/) and [rxjava](https://github.com/Netflix/RxJava) and came across the [following post](java.dzone.com/articles/creating-rxjava-observable) that described how to map rxjava observables from messages posted to akka actors.

Since my team works in java, I decided to try mapping the concept to java directly, but found that there was an issue. When I tried to have multiple subscribers listen on the stream I'd get an exception since more than one subscriber would send the "subscribe" message and try to modify the akka receive context.

I also wanted to make it easier to extend the actors to be able to process a piece of work, and then resubmit it for consumption by the observable.

## The subscribe command messages

First, let me show the commands we can send to the actors. This is just mapping the scala union type that the original blog post had. The `@Data` attribute is part of [project lombok](projectlombok.org/features/index.html) and auto creates an immutable class with a constructor and getters for your private final fields:

[java]  
package com.devshorts.rx;

import java.io.Serializable;

public class UnSubscribe implements Serializable {}  
[/java]

And

[java]  
package com.devshorts.rx;

import lombok.Data;  
import rx.functions.Action1;

import java.io.Serializable;

@Data  
public class Subscribe implements Serializable {  
 private final Action1 subscription;  
}  
[/java]

## Observable actor

Second, let me show the mapped observable actor. This is the abstract class that all observable actors should inherit from and it manages changing the default akka context to invoke receive messages on the supplied procedure. All this means is that when the actor gets a `Subscribe` it'll change what function it uses to receive messages to the one that is supplied by the subscribe object: `onRecieve` won't ever get called anymore. `unbecome` would undo that and set it back to the default handler.

[java]  
package com.devshorts.rx;

import akka.actor.UntypedActor;  
import akka.japi.Procedure;

public abstract class ObservableActor extends UntypedActor {

@Override  
 public void onReceive(final Object o) throws Exception {  
 if(o instanceof Subscribe){  
 System.out.println("Subscribed!");

// change the default 'onReceive' behavior to now be the anonymous class  
 // implementation. This means that all new requests will go to  
 // processMessage and returned to the observable as a transformation  
 getContext().become(new Procedure\<Object\>() {  
 @Override  
 public void apply(Object message) throws Exception {  
 if(message instanceof UnSubscribe){  
 getContext().unbecome();

System.out.println("Unsubscribed");  
 }  
 else{  
 Subscribe subscriber = (Subscribe)(o);

subscriber.getSubscription().call(processMessage(message));  
 }  
 }  
 });  
 }  
 else{  
 System.out.println("Default behavior " + o);  
 }

}

protected abstract Object processMessage(Object message);  
}  
[/java]

Notice the abstract method though. This is what I want all subsequent actors to implement and acts as the "do work" method.

Here's an actor that just re-dispatches its input:

[java]  
package com.devshorts.rx;

public class AkkEcho extends ObservableActor {  
 @Override  
 protected Object processMessage(Object message) {  
 return message;  
 }  
}  
[/java]

And here is one that modifies the input a little

[java]  
package com.devshorts.rx;

public class AkkaMapEcho extends ObservableActor {  
 @Override  
 protected Object processMessage(Object message) {  
 String m = (String)(message);

return m + " mapped!";  
 }  
}  
[/java]

## Creating the observable wrapper

Below is the observable wrapper. It creates a publish subject that handles incoming and outgoing messages, as well as taking care of instantiating only _one_ observable bound the actor. What is returned is now a safe consumable stream that multiple subscribers can read off of:

[java]  
package com.devshorts.rx;

import akka.actor.ActorRef;  
import rx.Observable;  
import rx.Subscriber;  
import rx.functions.Action1;  
import rx.subjects.PublishSubject;

public class ObservableUtil {

public static \<T\> Observable\<T\> fromActor(final ActorRef actor){  
 final PublishSubject\<T\> subj = PublishSubject.create();

Observable\<T\> observable = Observable.create(new Observable.OnSubscribe\<T\>() {  
 @Override  
 public void call(final Subscriber\<? super T\> subscriber) {

/\*\*  
 \* Create an initial subscribe method that modifies  
 \* the actors default behavior to proxy the request to the  
 \* subscribers 'onNext' function. This way  
 \* when someone posts to the actor, we intercept the actors RESPONSE  
 \* and pipe it into the subscribers work queue.  
 \*/  
 Subscribe msg = new Subscribe(new Action1\<T\>() {  
 @Override  
 public void call(T o) {  
 subscriber.onNext(o);  
 }  
 });

actor.tell(msg, ActorRef.noSender());  
 }  
 });

/\*\*  
 \* Create one subscriber to this actor observable and re-proxy the result  
 \* to the subject (this lets other people subscribe to the subject, and keeps  
 \* the akka observable from having to worry about managing who is substring to what  
 \* and de-muddles up the behavior modification code. this call also invokes the  
 \* subscribe command pattern above.  
 \*/  
 observable.subscribe(new Action1\<T\>() {  
 @Override  
 public void call(T o) {  
 subj.onNext(o);  
 }  
 });

/\*\*  
 \* Return the subject's observable stream for others to subscribe on  
 \*/  
 return subj.asObservable();  
 }  
}  
[/java]

## Using it

You can imagine a distributed system where you have actors and they are receiving messages, but you want to work on their output transformations or listen to them via the observable API.

This makes it really nice to have uniform time based event behaviors that you can leverage in your code. It no longer matters that the events are sourced from an akka actor, or if they are sourced from futures, or iterables, or whatever. They just exist, and you have subscribed to them.

Now let's check out a unit test that uses the actor. We'll have two observables that listen and capture events, and the test will also be responsible for posting values to the actor.

[java]  
/\*\*  
 \* Wrap an akka actor's behavior into an observable stream.  
 \*  
 \* Now your producer api is the actor, but your consumers can  
 \* manipulate the underlying event stream to create behaviors  
 \* @throws InterruptedException  
 \*/  
@Test  
public void AkkaObservable() throws InterruptedException {  
 final Object mutex = new Object();

final ActorRef actor = createActorOfType(AkkEcho.class);

final List\<String\> results = new ArrayList\<\>();  
 final List\<String\> distinctResults = new ArrayList\<\>();

final Observable\<String\> observable = ObservableUtil.fromActor(actor);

observable.subscribe(new Action1\<String\>() {  
 @Override  
 public void call(String o) {  
 System.out.println(o);

if(o.equals("done")){  
 synchronized (mutex){ mutex.notify(); }  
 }  
 else{  
 results.add(o);  
 }  
 }  
 });

observable.distinct().subscribe(new Action1\<String\>() {  
 @Override  
 public void call(String o) {  
 distinctResults.add(o);  
 }  
 });

actor.tell("foo", ActorRef.noSender());  
 actor.tell("foo", ActorRef.noSender());  
 actor.tell("foo", ActorRef.noSender());  
 actor.tell("bar", ActorRef.noSender());  
 actor.tell("done", ActorRef.noSender());

synchronized (mutex){ mutex.wait(); }

Assert.assertEquals(results, Arrays.asList("foo", "foo", "foo", "bar"));  
 Assert.assertEquals(distinctResults, Arrays.asList("foo", "bar", "done"));  
}

private ActorRef createActorOfType(Class\<? extends Actor\> clazz) {  
 ActorSystem system = ActorSystem.create("client");

return system.actorOf(Props.create(clazz), "rcv");  
}  
[/java]

Oh, and this is onoffswitch.net's 100th post! wooo!

