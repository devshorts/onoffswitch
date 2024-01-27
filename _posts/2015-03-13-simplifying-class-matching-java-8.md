---
layout: post
title: Simplifying class matching with java 8
date: 2015-03-13 23:51:30.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- akka
- combinators
- java
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560443277;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4627;}i:1;a:1:{s:2:"id";i:4456;}i:2;a:1:{s:2:"id";i:4629;}}}}

permalink: "/2015/03/13/simplifying-class-matching-java-8/"
---
I'm knee deep in akka these days and its a great queueing framework, but unfortunately I'm stuck using java and not able to use scala (business decisions, not mine!) so pattern matching on incoming untyped events can be kind of nasty.

You frequently see stuff like this in receive methods:

```java
  
public void onReceive(Object message){  
 if(message instanceof Something){

}  
 else if (message instanceof SomethingElse){

}  
 .. etc  
}  

```

And while that technically works, I really hate it because it promotes a monolothic function doing too much work. It also encourages less disciplined devs to put logic into the if block. While this is fine for a few checks, what happens when you need to dispatch 10, or 20 different types? It's not uncommon in actor based systems to have lots of small message types.

Also, because akka gives you your object as a type erased Object you can't use normal dispatching mechanisms like overloaded functions. And to complicate things even more, you can't really use a visitor pattern without conflating business logic into your data access objects.

In reality, all I want is that my receive function should act as dispatcher, doing the right type checking on your object and executing a function that does the specific work. Now I can write small functions for each particular message type and keep that logic encapsulated and composable.

Thankfully with java 8 and lambadas we can create some simple combinator style executors that let us do stuff like this:

```java
  
@Override public void onReceive(final Object message) throws Exception {  
 match().with(Payload.class, this::handlePayload)  
 .with(ResizeWorkLoadMessage.class, this::processResize)  
 .with(HeartBeat.class, this::heartBeat)  
 .fallthrough(i -\> logger.with(i).warn("Unknown type called to actor, cannot route"))  
 .exec(message);  
}

private void handlePayload(Payload payload){  
 // ...  
}

private void processResize(ResizeWorkLoadMessage resizeWorkload){  
 // ...  
}  

```

Now we have a simple cast matcher that checks a raw object type, does a monadic check to see which matcher succeeds (going from top down with priority) and if nothing matches executes the fallthrough.

Building something like this is pretty trivial. It's just a combinator that captures the current state and delegates to the next state if the current cast doesn't succeed:

```java
  
import java.util.function.BiFunction;  
import java.util.function.Consumer;

public class ClassMatcher {

private final BiFunction\<Object, Consumer\<Object\>, Boolean\> binder;

private ClassMatcher(BiFunction\<Object, Consumer\<Object\>, Boolean\> next) {  
 this.binder = next;  
 }

public void exec(Object o) {  
 binder.apply(o, null);  
 }

public \<Y\> ClassMatcher with(final Class\<Y\> targetClass, final Consumer\<Y\> consumer) {  
 return new ClassMatcher((obj, next) -\> {

if (binder.apply(obj, next)) {  
 return true;  
 }

if (targetClass.isAssignableFrom(obj.getClass())) {  
 final Y as = (Y) obj;

consumer.accept(as);

return true;  
 }

return false;  
 });  
 }

public ClassMatcher fallthrough(final Consumer\<Object\> consumer) {  
 return new ClassMatcher((obj, next) -\> {

if (binder.apply(obj, next)) {  
 return true;  
 }

consumer.accept(obj);

return true;

});  
 }

public static ClassMatcher match() {  
 return new ClassMatcher((a, b) -\> false);  
 }  
}  

```

## Performance

Reddit seemed to be obsessed about the perf costs with this implementation. I spun up JMH and gave it a whirl to compare the cost of this vs if statements.

You do pay a small performance penalty for this higher level abstraction but I think its a small price to pay. Almost any higher abstraction pays a penalty of some sort. In JMH benchmarking methods that used an if tree took a pretty constant 20-50 nanoseconds to complete, and ones using a matcher took about 2-4 times longer (around 90 nanoseconds for a matcher of 4 cases).

Caching an instance of your dispatcher cut the perf time in half, and when you build out lots of match statements it makes a more noticable difference (20 match statements recreated each time was 500 nanoseconds and cached it was 150 nanoseconds). Downside to caching is that you can't close over anything and create adhoc functions inline, but if you use it as a pure dispatcher then caching is fine.

Just as an example a simple cacher:

```java
  
import java.util.function.Supplier;

public class ClassMatchCache {  
 private ClassMatcher matcher;

public ClassMatcher cache(Supplier\<ClassMatcher\> matchFactory) {  
 if(matcher == null){  
 matcher = matchFactory.get();  
 }

return matcher;  
 }  
}  

```

And you can now use it

```java
  
public class EventHandler {  
 private ClassMatchCache mainDispatcher = new ClassMatchCache();

public void dispatch(Object o){  
 mainDispatcher.cache(  
 () -\> match().with(Bar.class, this::bar)  
 .with(Biz.class, this::biz)  
 .with(Baz.class, this::baz)  
 .with(Foo1.class, this::foo)  
 .with(Foo2.class, this::foo)  
 .with(Foo3.class, this::foo)  
 .with(Foo4.class, this::foo)  
 .with(Foo5.class, this::foo)  
 .with(Foo13.class, this::foo)  
 .fallthrough(this::fallthrough))  
 .exec(o);  
 }  
}  

```

Or use whatever caching mechanism works for you.

But in the end, do 100 nanoseconds matter to you to jump through all these hoops? To put it in perspective the following statement

```java
  
IntStream.range(0, 1000).map(i -\> i + 1).sum();  

```

Takes 400 times longer, averaging 4000 nanoseconds. And even then, what's 4000 nanoseconds? Thats 4 microseconds, and .004 milliseconds. In the large scope of real projects this is insignificant.

The rule of thumb with all optimizations is don't prematurely optimize. If you really had a perf hit, switch your code to if statements.

