---
layout: post
title: Converting akka scala futures to java futures
date: 2015-05-07 01:13:48.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- akka
- async
- futures
- java
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559708751;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4456;}i:1;a:1:{s:2:"id";i:4627;}i:2;a:1:{s:2:"id";i:4394;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/05/07/converting-akka-scala-futures-java-futures/"
---
Back in akka land! I'm using the ask pattern to get results back from actors since I have a requirement to block and get a result (I can't wait for an actor to push at a later date). Thats fine, but converting from scala futures to java completable futures is a pain. I also, (like mentioned in another post) want to make sure that my async responses capture and set the MDC for proper logging.

My final usage should look something like:

[java]  
private \<Response, Request\> Future\<Response\> askActorForResponseAsync(Request source) {  
 final FiniteDuration askTimeout = new FiniteDuration(config.getAskForResultTimeout().toMillis(), TimeUnit.MILLISECONDS);

final Timeout timeout = new Timeout(askTimeout);

final scala.concurrent.Future\<Object\> ask = Patterns.ask(master.getActor(), new PersistableMessageContext(source), timeout);

return FutureConverter.fromScalaFuture(ask)  
 .executeOn(actorSystem.dispatcher())  
 .thenApply(i -\> (Response) i);  
}  
[/java]

The idea is that I'm going to translate a scala future with a callback into a completable future java promise.

Next up, the future converter:

[java]  
public class FutureConverter {  
 public static \<T\> FromScalaFuture\<T\> fromScalaFuture(scala.concurrent.Future\<T\> future) {  
 return new FromScalaFuture\<\>(future);  
 }  
}  
[/java]

This is just an entrypoint into a new class that can give you a nice fluent interface to provide the execution context.

Next, a class whose job is to create an akka callback and convert it into a completable future.

[java]  
import scala.concurrent.ExecutionContext;  
import scala.concurrent.Future;

import java.util.concurrent.CompletableFuture;

public class FromScalaFuture\<T\> {

private final Future\<T\> future;

public FromScalaFuture(Future\<T\> future) {  
 this.future = future;  
 }

public CompletableFuture\<T\> executeOn(ExecutionContext context) {  
 final CompletableFuture\<T\> completableFuture = new CompletableFuture\<\>();

final AkkaOnCompleteCallback\<T\> completer = AkkaCompletionConverter.\<T\>createCompleter((failure, success) -\> {  
 if (failure != null) {  
 completableFuture.completeExceptionally(failure);  
 }  
 else {  
 completableFuture.complete(success);  
 }  
 });

future.onComplete(completer.toScalaCallback(), context);

return completableFuture;  
 }  
}  
[/java]

And finally another guy whose job it is to translate java functions into akka callbacks:

[java]  
import akka.dispatch.OnComplete;

@FunctionalInterface  
public interface AkkaOnCompleteCallback\<T\> {  
 OnComplete\<T\> toScalaCallback();  
}  
[/java]

[java]  
import akka.dispatch.OnComplete;  
import org.slf4j.MDC;

import java.util.Map;  
import java.util.function.BiConsumer;

public class AkkaCompletionConverter {  
 /\*\*  
 \* Handles closing over the mdc context map and setting the responding future thread with the  
 \* previous context  
 \*  
 \* @param callback  
 \* @return  
 \*/  
 public static \<T\> AkkaOnCompleteCallback\<T\> createCompleter(BiConsumer\<Throwable, T\> callback) {  
 return () -\> {

final Map\<String, String\> oldContextMap = MDC.getCopyOfContextMap();

return new OnComplete\<T\>() {  
 @Override public void onComplete(final Throwable failure, final T success) throws Throwable {  
 // capture the current threads context map  
 final Map\<String, String\> currentThreadsContext = MDC.getCopyOfContextMap();

// set the closed over context map  
 if(oldContextMap != null) {  
 MDC.setContextMap(oldContextMap);  
 }

callback.accept(failure, success);

// return the current threads previous context map  
 if(currentThreadsContext != null) {  
 MDC.setContextMap(currentThreadsContext);  
 }  
 }  
 };  
 };  
 }  
}  
[/java]

