---
layout: post
title: Building LINQ in Java pt 2
date: 2014-02-24 08:00:16.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- iterator
- java
- linq
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1557352026;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4316;}i:1;a:1:{s:2:"id";i:3367;}i:2;a:1:{s:2:"id";i:3497;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2014/02/24/building-linq-java-pt-2/"
---
In my [last post](http://onoffswitch.net/building-linq-java/ "Building LINQ in Java") I discussed building a static class that worked as the fluent interface exposing different iterator sources that provide transformations. For 1:1 iterators, like take, skip, while, for, nth, first, last, windowed, etc, you just do whatever you need to do internally in the iterator by manipulating the output the stream.

But if you want to do a transformation like a map, you need to project the input source to something else. Now the base class each iterator inherits from isn't good enough. But thankfully we can just add another generic parameter and create a new mappable base class that can handle transformations.

The following iterator handles projecting from a source to a result type and yields an enumerable iterator of the result type.

[java]  
package com.devshorts.enumerable.iterators;

import java.util.function.Function;

public class MapIterator\<TSource, TResult\> extends EnumerableIterator\<TResult\> {

private Function\<TSource, TResult\> projection;

/\*\*\*  
 \* Need this constructor for flatMap  
 \* @param input  
 \*/  
 protected MapIterator(Iterable input){  
 super(input);

// by default the projection is the id function  
 this.projection = i -\> (TResult)i;  
 }

public MapIterator(Iterable\<TSource\> source, Function\<TSource, TResult\> projection) {  
 this(source);

this.projection = projection;  
 }

@Override  
 public boolean hasNext() {  
 return source.hasNext();  
 }

@Override  
 public TResult next() {  
 return projection.apply((TSource)source.next());  
 }  
}  
[/java]

It's exposed in the main base class by passing in a projection function

[java]  
public \<TResult\> Enumerable\<TResult\> map(Function\<TSource, TResult\> mapFunc){  
 return enumerableWithIterator(source -\>  
 new MapIterator\<TSource, TResult\>(source, i -\> mapFunc.apply(i)));  
}

private \<TResult\> Enumerable\<TResult\> enumerableWithIterator(Function\<Iterable\<TSource\>, Iterator\<TResult\>\> generator){  
 return new Enumerable\<\>(\_ig -\> generator.apply(this));  
}  
[/java]

So enumerableWithIterator takes a function that returns an iterable, and wraps a new Enumerable fluent type to wrap the current iterator source (this). The "\_ig" parameter is ignored, hence the name.

## Flat Map

Now that we have a basic projection iterator, we can also implement a flat map. Flat map is the functional term for what some languages called "collect" or "selectMany". It takes a list of lists and yields each element in all the lists. Basically it flattens everything.

First lets check the flat map iterator. We need to consume each underlying list, buffer it, and then yield each element of each list when requested. If we've consumed the entire buffered list we need to get the next list.

Note how the flat map iterator inherits from the map iterator. The reason for this is that if we want to project from one type to another we need to inherit from a class that exposes two generic types. The basic EnumerableIterator only exposes one generic. We could have inherited from that as well and added the extra generics, but I think it was nicer to group maps in a map inheritance tree, and everything else under the basic class.

[java]  
package com.devshorts.enumerable.iterators;

import java.util.List;  
import java.util.function.Function;

/\*\*  
 \* Created with IntelliJ IDEA.  
 \* User: anton.kropp  
 \* Date: 11/11/13  
 \* Time: 1:04 PM  
 \* To change this template use File | Settings | File Templates.  
 \*/  
public class FlatMapIterator\<TSource, TResult\> extends MapIterator\<TSource, TResult\> {  
 private Function\<TSource, List\<TResult\>\> flatMapper;

public FlatMapIterator(Iterable\<TSource\> source, Function\<TSource, List\<TResult\>\> flatMapper) {  
 super(source);  
 this.flatMapper = flatMapper;  
 }

private List\<TResult\> \_bufferedResult;  
 private Integer idx = 0;

@Override  
 public boolean hasNext() {  
 if(\_bufferedResult == null && source.hasNext()){  
 \_bufferedResult = flatMapper.apply((TSource)source.next());

idx = 0;

return true;  
 }

if(\_bufferedResult != null){  
 if(idx \< \_bufferedResult.size()){  
 return true;  
 }

\_bufferedResult = null;

return hasNext();  
 }

return false;  
 }

public TResult next() {  
 TResult item = \_bufferedResult.get(idx);  
 idx++;  
 return item;  
 }  
}  
[/java]

## Other fun iterators

Since we have basic sequence manipulators and element transformations, we can do more complex LINQ type things. I like intersperse and windowed, since they are common in functional programming. Intersperse puts in an element inbetween each other element in a sequence. Imagine you have a list of characters ['a, 'b', 'c'] and you want to add in a comma in between them all. You can intersperse it with a comma and get ['a, ',','b',',','c'].

Technically intersperse is a subset of intercalate (terms taken from Haskells Data.List package). Intercalate will put in the elements of another list inbetween the elements of the first list. To get intersperse you pass in an array of size one:

[java]  
public class IntercalateIterator\<TSource\> extends EnumerableIterator\<TSource\> {  
 private List\<TSource\> intercalator;

public IntercalateIterator(Iterable\<TSource\> input, List\<TSource\> intercalator) {  
 super(input);  
 this.intercalator = intercalator;  
 }

private boolean intercalate = false;  
 private int idx = 0;

@Override  
 public boolean hasNext(){  
 if(intercalate && source.hasNext()){  
 return idx \< intercalator.size();  
 }

return source.hasNext();  
 }

@Override  
 public TSource next(){  
 TSource n;  
 if(intercalate){  
 n = intercalator.get(idx);

idx++;

if(idx == intercalator.size()){  
 intercalate = false;  
 }  
 }  
 else{  
 idx = 0;

n = source.next();

intercalate = true;  
 }

return n;  
 }

}  
[/java]

And check windowed. This yields a list of lists where each list is a sliding window of size N across the source list. If you a have a list

[code]  
1;2;3;4  
[/code]

And you apply a window of size 2, you will get

[code]  
[[1;2], [2;3], [3;4]]  
[/code]

This can be pretty handy in some scenarios

[java]  
package com.devshorts.enumerable.iterators;

import java.util.Iterator;  
import java.util.LinkedList;  
import java.util.List;  
import java.util.Queue;

public class WindowedIterator\<TSource\> extends MapIterator\<TSource, List\<TSource\>\> {  
 private final int windowSize;  
 private boolean windowSeeded;  
 List\<TSource\> queue = new LinkedList\<\>();  
 List\<TSource\> next = new LinkedList\<\>();

public WindowedIterator(Iterable\<TSource\> input, int windowSize) {  
 super(input);  
 this.windowSize = windowSize;  
 }

@Override  
 public boolean hasNext(){  
 if(!windowSeeded){  
 seedWindow();

windowSeeded = true;  
 }

return next.size() \> 0;  
 }

@Override  
 public List\<TSource\> next(){  
 queue = new LinkedList\<\>(next);

nextWindow();

if(next.size() != windowSize){  
 next.clear();  
 }

return (List\<TSource\>)(queue);  
 }

private void nextWindow(){  
 next.remove(0);

if(it().hasNext()){  
 next.add(it().next());  
 }  
 }

private void seedWindow(){  
 int window = windowSize;  
 while(it().hasNext() && window \> 0){  
 next.add(it().next());  
 window--;  
 }  
 }

private Iterator\<TSource\> it(){  
 return (Iterator\<TSource\>)(source);  
 }

}  
[/java]

## Using it

Using these features is now really easy, here are some sample tests that demonstrate our new iterators

[java]  
@Test  
public void FlatMap(){  
 assertEquals(asList("5", "4", "3", "2", "1"),  
 Enumerable.init(asList(asList("5"), asList("4"), asList("3"), asList("2"), asList("1")))  
 .flatMap(i -\> i)  
 .map(i -\> i.toString())  
 .toList());  
}

@Test  
public void Map(){  
 assertEquals(asList("5", "4", "3", "2", "1"),  
 Enumerable.init(asList(5, 4, 3, 2, 1))  
 .map(i -\> i.toString())  
 .toList());  
}

@Test  
public void Windowed(){  
 assertEquals(asList(asList(1, 2), asList(2, 3), asList(3, 4)),  
 Enumerable.init(asList(1, 2, 3, 4))  
 .windowed(2)  
 .toList());  
}

[/java]

