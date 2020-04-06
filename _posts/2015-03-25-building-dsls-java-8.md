---
layout: post
title: Handling subclassed constraints with a DSL in java 8
date: 2015-03-25 01:42:01.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dsl
- java
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554060671;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4629;}i:2;a:1:{s:2:"id";i:1587;}}}}

permalink: "/2015/03/25/building-dsls-java-8/"
---
I really like doing all of my domain modeling with clean DSL's (domain specific languages). Basically I want my code to read like a sentence, and to hide all the magic behind things. When things read clearly even a non professional can determine if something is wrong. The ideal scenario is to have your code read like pseudocode since nobody really cares what the internals are, what matters is your general solution.

I found myself recently in a scenario where I have some methods that do work, and they all return a subtype of a response root. Something like:

[java]  
class Response extends ResponseRoot {}

class Worker{  
 Response doWork(Request request) { // }  
}  
[/java]

And I need to create a dynamic callback given the context of the request. So basically I want

[java]  
Consumer\<ResponseRoot\> callback = generateCallback(request);

Response response = doWork(request);

callback.accept(response);  
[/java]

However I am going to have a lot of this same boilerplate for many different kinds of requests. In a previous post I mentioned the `match` on runtime objects and this is the next phase of that scenario: take an untyped object, cast it to see what kind of object it is, depending on the object do a strongly typed method and then execute the callback.

I could create a bunch of methods that just create a new client, do the work, then issue the callback but thats no fun. Why not something like

[java]  
@Override public void processAsyncable(final Object input) throws Exception {  
 final Consumer\<ResponseRoot\> complete = getCompletionCallback(input);

match().with(SubRequest.class, afterDoing(this::subRequest, then(complete)))  
 .exec(input);  
}

private SubRequestResponse subRequest(final SubRequest availabilityCheckEvent) throws Throwable {  
 // ...  
}  
[/java]

What is the `afterDoing` and the `then(complete)`? Its reminiscent of junit matchers and their DSL.

[java]  
import org.jooq.lambda.fi.util.function.CheckedFunction;

import java.util.function.Consumer;

public class CompletionCallback {

public static \<TInput, TRoot, TResponse extends TRoot\> Consumer\<TInput\> afterDoing(CheckedFunction\<TInput, TResponse\> function, ThenVerb\<TRoot\> verb) {  
 return input -\>  
 {  
 TRoot result = null;  
 try {  
 result = function.apply(input)  
 }  
 catch (Throwable throwable) {  
 throw new RuntimeException(throwable);  
 }

verb.accept(result);  
 };  
 }

public static \<T\> ThenVerb\<T\> then(Consumer\<T\> consumer) {  
 return new ThenVerb\<\>(consumer);  
 }  
}

class ThenVerb\<Y\> {  
 private final Consumer\<Y\> consumer;

public ThenVerb(Consumer\<Y\> consumer) {  
 this.consumer = consumer;  
 }

public void accept(Y item){  
 consumer.accept(item);  
 }  
}  
[/java]

The idea here is to try and model the "_Response subclasses RootResponse and I have a function that can take that subclassed item but I only know about the root_" statement. This is why there are 3 generics. By defining the verb and making it generic we can now give the verb the generic constraint necessary to model this `Response extends ResponseRoot` constraint. The first parameter gets the input function that generates the value to pass to the completor. The verb just wraps the consumer which is the final completion object.

While it looks like it's a lot of extra work, what I like about this pattern is that your code reads now like a sentence.

