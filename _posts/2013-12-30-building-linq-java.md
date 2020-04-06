---
layout: post
title: Building LINQ in Java
date: 2013-12-30 08:00:34.000000000 -08:00
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
- lambda
- lazy
- linq
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559686498;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4306;}i:1;a:1:{s:2:"id";i:4355;}i:2;a:1:{s:2:"id";i:4862;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/12/30/building-linq-java/"
---
Now that Java 8 has lambdas, I decided to check out what kind of lazy collection support their streams functionality had. It had some cool stuff, like

- map
- filter
- flatMap
- distinct
- sorted
- limit (i.e. take)
- skip
- reduce (i.e fold)
- min
- max
- any
- all
- generate (for building infinite lists)

Not too bad. But, there are some problems, as far as I see it. First, you can't extend it. With .NET they solved the lambda problem with extension methods, but Java oddly chose "defender" methods, which lets you safely update an interface declaration, but ONLY the author can do it, not any consumer. That sucks. I want other fun functions like

- first
- last
- nth
- windowed
- intersperse
- intercalate
- tails
- intersect
- zip
- groupRuns

And pretty much any other F#/Haskell list function. Why not? If we're going to go with lazy evaluated lists we might as well have all the fun that comes with them. There is another problem with the Java 8 streams API, unless I'm mistaken, there is no way to _end_ an infinite stream generated using the `generate` function. That really sucks since not every lazy generated stream is infinite.

So, I decided to try my hand and rebuilding LINQ in Java 8. For the impatient, the full source, tests, and benchmarks are available at my [github](https://github.com/devshorts/JEnumerable).

## Iterator chains

The basic idea here is to create an iterator for each type of processing you want to do. If you want to do a map function, you should create an iterator that wraps a source. When the iterator consumes from the underlying source it will emit a projected element. How do you do this? Well, you can use a fluent API that returns a new enumerable class that wraps a specific iterator. For example

[java]  
private Function\<Iterable\<TSource\>, Iterator\<TSource\>\> iteratorGenerator;

public static \<TSource\> Enumerable\<TSource\> init(Iterable\<TSource\> source){  
 return new Enumerable\<\>(\_ig -\> new EnumerableIterator\<\>(source));  
}

protected Enumerable(Function\<Iterable\<TSource\>, Iterator\<TSource\>\> iteratorGenerator) {  
 this.iteratorGenerator = iteratorGenerator;  
}

@Override  
public Iterator\<TSource\> iterator() {  
 return iteratorGenerator.apply(this);  
}

// The underlying iterator

public class EnumerableIterator\<TSource\> implements Iterator\<TSource\> {  
 protected Iterator\<TSource\> source;  
 private Iterable\<TSource\> input;

public EnumerableIterator(Iterable\<TSource\> input){  
 this.input = input;

reset();  
 }

protected void reset(){  
 source = input.iterator();  
 }

@Override  
 public boolean hasNext() {  
 return source.hasNext();  
 }

@Override  
 public TSource next() {  
 return (TSource)source.next();  
 }  
}  
[/java]

Lets first look at the underlying iterator. It does nothing other than iterate over the source. That's pretty simple.

The thing that wraps it is a `Enumerable` class that takes a function that, when given an Iterable, returns a new iterator. Then it just exposes that iterator. In general, thats the whole thing. The iterator is wrapped in a function so that we can request a new iterator each time someone tries to iterate over this enumerable. That matches what .NET does; if someone tries to re-iterate an enumerable you get a new iterator and start over.

## Take

Let's look at a simple iterator that takes only N elements.

[java]  
public class TakeIterator\<TSource\> extends EnumerableIterator\<TSource\> {  
 private int takeNum;

public TakeIterator(Iterable\<TSource\> results, int n) {  
 super(results);  
 takeNum = n;  
 }

@Override  
 public boolean hasNext() {  
 return source.hasNext() && takeNum \> 0;  
 }

@Override  
 public TSource next(){  
 takeNum--;  
 return source.next();  
 }  
}  
[/java]

And to create an instance of enumerable that uses this

[java]  
public Enumerable\<TSource\> take(int n){  
 return enumerableWithIterator(source -\> new TakeIterator\<\>(source, n));  
}

private \<TResult\> Enumerable\<TResult\> enumerableWithIterator(Function\<Iterable\<TSource\>, Iterator\<TResult\>\> generator){  
 return new Enumerable\<\>(\_ig -\> generator.apply(this));  
}  
[/java]

Basically I return a new enumerable with a lazy evaluated function that gives the iterator its underlying Iterator source. By returning a new enumerable each time we can effectively chain the iterators together. Nothing is evaluated until someone tries to get the next value.

