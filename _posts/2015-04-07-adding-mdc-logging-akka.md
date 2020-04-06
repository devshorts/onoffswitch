---
layout: post
title: Adding MDC logging to akka
date: 2015-04-07 01:05:20.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- akka
- java
- MDC
- slf4j
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560404400;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4596;}i:1;a:1:{s:2:"id";i:4629;}i:2;a:1:{s:2:"id";i:4593;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/04/07/adding-mdc-logging-akka/"
---
I've mentioned before, but I'm working heavily in a project that is leveraging akka. I am really enjoying the message passing model and so far things are great, but tying in an MDC for the SLFJ logging context proved complicated. I had played with the custom executor model described [here](http://yanns.github.io/blog/2014/05/04/slf4j-mapped-diagnostic-context-mdc-with-play-framework/) but hadn't attempted the akka custom dispatcher.

I was thinking that a custom dispatcher would work great to pass along the MDC since then you'd never have to think about it, but unfortunately I couldn't get it to work. Akka kept failing to instantiate the dispatcher. I was also worried about configuration data and possible tuning that you might lose giving akka your own dispatcher configurator.

So, given that I wasn't quite sure what to do. What I ended up with however was a little extra work but turned out well. I went with an augmented dispatcher/subscriber model. Basically for every event that I send out I wrap it in a `PersistentableMessage` which traps the fields of the MDC that I care about, and then on any actor I have them subclass a custom logging base class that pops out the persistent message container, sets the MDC, and gives the actor the underlying message.

For my project we're tracking everything with what we call a `CorrelationId` which is just a UUID.

## The message wrapper

[java]  
@Data  
public class PersistableMessageContext implements CorrelationIdGetter, CorrelationIdSetter {  
 private final Object source;

private UUID correlationId;

public PersistableMessageContext(Object source){  
 this.source = source;

try {  
 final String s = MDC.get(FilterAttributes.CORR\_ID);

setCorrelationId(UUID.fromString(s));  
 }  
 catch(Throwable ex){}  
}  
[/java]

This is the message that I want to pass around. By containing its source correlation ID it can later be used to set the context when its being consumed

## The actor base

I now have all my actors subclass this class

[java]  
public abstract class LoggableActor extends UntypedActor {  
 @Override public void onReceive(final Object message) throws Exception {  
 Boolean wasSet = false;

if (CorrelationIdGetter.class.isAssignableFrom(message.getClass())) {  
 final UUID correlationId = ((CorrelationIdGetter) message).getCorrelationId();

if (correlationId != null) {  
 MDC.put(FilterAttributes.CORR\_ID, correlationId.toString());

wasSet = true;  
 }  
 }

if (message instanceof PersistableMessageContext) {  
 onReceiveImpl(((PersistableMessageContext) message).getSource());  
 }  
 else {  
 onReceiveImpl(message);  
 }

if(wasSet) {  
 MDC.remove(FilterAttributes.CORR\_ID);  
 }  
 }

public abstract void onReceiveImpl(final Object message) throws Exception;  
}  
[/java]

This lets me pass in anything that implements a `CorrelationIdGetter` and if it happens to also be a persisted message, pop out the inner message.

## Sending out messages

Now the big issue here is to make sure that we are consistent in publishing messages. This means using routers, broadcasts, etc, all have to make sure to push out a message wrapped in a persistent container. To help make that easier I created a few augmented akka publisher classes. Below is a class with static methods (to make it easy to import) that wrap an actor ref or a router.

[java]  
import akka.actor.ActorRef;  
import akka.routing.Broadcast;  
import akka.routing.Router;

/\*\*  
 \* Utilitiy to provide context propagation on akka messages  
 \*/  
public class AkkaAugmenter {

public static AkkaAugmentedActor wrap(ActorRef src) {  
 return new AkkaAugmentedActor(){  
 @Override public void tell(final Object msg, final ActorRef sender) {  
 final PersistableMessageContext persistableMessageContext = new PersistableMessageContext(msg);

src.tell(persistableMessageContext, sender);  
 }

@Override public ActorRef getActor() {  
 return src;  
 }  
 };  
 }

public static AkkaAugmentedRouter wrap(Router src) {  
 return new AkkaAugmentedRouter() {  
 @Override public void route(final Object msg, final ActorRef sender) {  
 final PersistableMessageContext persistableMessageContext = new PersistableMessageContext(msg);

src.route(persistableMessageContext, sender);  
 }

@Override public void broadcast(final Object msg, final ActorRef sender) {  
 final PersistableMessageContext persistableMessageContext = new PersistableMessageContext(msg);

src.route(new Broadcast(persistableMessageContext), sender);  
 }

@Override public Router getRouter() {  
 return src;  
 }  
 };  
 }  
}  
[/java]

The augmented actor:

[java]  
public interface AkkaAugmentedActor {  
 void tell(Object msg, ActorRef sender);

ActorRef getActor();  
}  
[/java]

And the augmented router:

[java]  
public interface AkkaAugmentedRouter {  
 void route(Object msg, ActorRef sender);

void broadcast(Object msg, ActorRef sender);

Router getRouter();  
}  
[/java]

## Conclusion

And now all I need to do is to wrap a default actor or router give to me by akka. From here on out all messages are auto wrapped and my MDC is properly propagated. While I would have liked to not rely on convention this way, at least I made it simple once you've made the right types.

