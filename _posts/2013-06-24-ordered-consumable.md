---
layout: post
title: Ordered Consumable
date: 2013-06-24 08:00:23.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- data structure
- enumerable
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554944619;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4892;}i:1;a:1:{s:2:"id";i:3640;}i:2;a:1:{s:2:"id";i:6;}}}}

permalink: "/2013/06/24/ordered-consumable/"
---
I had the need for a specific collection type where I would only ever process an element once, but be able to arbitrarily jump around and process different elements. Once a jump happened, the elements would be processed in circular order: continue to the end, then loop around to the beginning and process any remaining items.

The use case that prompted this is I have an image generator that creates snapshots out of a video on demand. However, I need to be able to seek in the video and create snapshots at the seeked point in time. Once all the snapshots are created I don't need to create them again, it's just a one time processing, but the snapshot generation has to follow the users actions. This also means that if a user seeks to a point in time where a snapshot was already generated, the snapshot generation doesn't need to re-process that image, but it should start processing any pending images linearlly in time nearby where the user went to.

For example, imagine you have a video that's an hour long. You start creating images at time 0, then time 1, then time 2, etc. At time 10, the user seeks to time 390 (6.5 minutes). We should jump and create image 390, then start 391, then 392, etc. If a user goes back to time 0, we don't need to process time 0 again, but should jump to time 11 (0 through 10 were already processed). Then time 12, then time 13, etc.

Here's a simplified example of what I want. The arrow points to the item to be processed.

Start with a regular list

[![start](http://onoffswitch.net/wp-content/uploads/2013/05/start-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/start.jpg)

Consume 1, move to 2.

[![2](http://onoffswitch.net/wp-content/uploads/2013/05/2-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/2.jpg)

Consume 2, but then jump to 13.

[![shiftTo13](http://onoffswitch.net/wp-content/uploads/2013/05/shiftTo13-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/shiftTo13.jpg)

Consume 13, reach the end and wrap around. 1, and 2 are already processed, next to process is 3

[![3](http://onoffswitch.net/wp-content/uploads/2013/05/3-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/3.jpg)

Consume 3, move to 4

[![4](http://onoffswitch.net/wp-content/uploads/2013/05/4-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/4.jpg)

Consume 4, move to 5

[![5](http://onoffswitch.net/wp-content/uploads/2013/05/5-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/5.jpg)

Jump to 10, consume, and move to 11

[![10](http://onoffswitch.net/wp-content/uploads/2013/05/101-1024x141.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/101.jpg)

Until the whole list is processed. Once the list is processed you can't consume anything else.

Here is a unit test to demonstrate in code:

[csharp]  
[Test]  
public void TestConsuming()  
{  
 var l = new List\<int\> { 1, 2, 3, 4, 5, 6, 7, 8, 9 };

var consumable = new OrderedConsumable\<int\>(l);

consumable.SetAsHead(item =\> item == 8);

var sort = consumable.Take(4).ToList();

Assert.IsTrue(sort.First() == 8);  
 Assert.IsTrue(sort.Last() == 2);

consumable.SetAsHead(item =\> item == 8);

sort = consumable.Take(40).ToList();

Assert.IsTrue(sort.First() == 3);  
 Assert.IsTrue(sort.Last() == 7);  
}  
[/csharp]

The easiest way to implement this was to wrap a list with a custom IEnumerable. The underlying enumerator can track if an item is processed or not, and if so find the next item to emit.

Here's the IEnumerable

[csharp]  
public class OrderedConsumable\<T\> : IEnumerable\<T\>  
{  
 public IList\<T\> List { get; set; }

private OrderedConsumableEnumerator\<T\> Enumerator { get; set; }

public OrderedConsumable(IList\<T\> list)  
 {  
 List = list;

Enumerator = new OrderedConsumableEnumerator\<T\>(List);  
 }

public bool SetAsHead(Func\<T, bool\> selector)  
 {  
 var item = List.FirstOrDefault(selector);

var indx = List.IndexOf(item);

if (indx != -1)  
 {  
 var alreadyProcessedIndex = Enumerator.HasProcessed(indx);

Enumerator.SetIndexTo(indx);

return !alreadyProcessedIndex;  
 }

return false;  
 }

public void Unmark(T item)  
 {  
 var idx = List.IndexOf(item);

if (idx != -1)  
 {  
 Enumerator.ReProcess(idx);  
 }  
 }

public void Reset()  
 {  
 Enumerator.Reset();  
 }

public Boolean CompletelyConsumed  
 {  
 get { return Enumerator.CompletelyConsumed; }  
 }

public IEnumerator\<T\> GetEnumerator()  
 {  
 return Enumerator;  
 }

IEnumerator IEnumerable.GetEnumerator()  
 {  
 return GetEnumerator();  
 }  
}  
[/csharp]

It's basically just a wrapper over the enumerator. The enumerator does all the real work:

[csharp]  
public class OrderedConsumableEnumerator\<T\> : IEnumerator\<T\>  
{  
 private readonly object \_locker = new object();

private IList\<T\> List { get; set; }

private int Index { get; set; }

private List\<Boolean\> Processed { get; set; }

private int Length { get; set; }

private int ConsumedCount { get; set; }

public OrderedConsumableEnumerator(IList\<T\> list)  
 {  
 List = list;  
 Index = 0;

ConsumedCount = 0;

Length = List.Count;

Processed = List.Select(f =\> false).ToList();

}

public void Dispose()  
 {

}

public bool MoveNext()  
 {  
 lock (\_locker)  
 {  
 if (!CompletelyConsumed)  
 {  
 while (Processed[Index % Length])  
 {  
 Index++;

if (Index \>= Length)  
 {  
 Index = 0;  
 }  
 }

return true;  
 }

return false;  
 }  
 }

public void Reset()  
 {  
 lock (\_locker)  
 {  
 Index = -1;

Processed = Processed.Select(i =\> false).ToList();  
 }  
 }

public void SetIndexTo(int indx)  
 {  
 lock (\_locker)  
 {  
 Index = indx;  
 }  
 }

public T Current  
 {  
 get  
 {  
 lock (\_locker)  
 {  
 Processed[Index] = true;  
 ConsumedCount++;  
 return List[Index];  
 }  
 }  
 }

object IEnumerator.Current  
 {  
 get { return Current; }  
 }

public Boolean CompletelyConsumed  
 {  
 get { return ConsumedCount == Length; }  
 }

public void ReProcess(int idx)  
 {  
 lock (\_locker)  
 {  
 Processed[idx] = false;  
 ConsumedCount--;  
 }  
 }

public bool HasProcessed(int indx)  
 {  
 lock (\_locker)  
 {  
 return Processed[indx];  
 }  
 }  
}  
[/csharp]

The enumerator uses a boolean array (the same size as the input list) to track if something is processed or not. When someone calls `Current` it marks the current index as processed and returns the value. When `MoveNext` is executed, we just need to find the next unprocessed element in the processed list and set the underlying index to that element.

While using a linked list might have been more space and time efficient to track what is next (since iterating over the while loop if the array gets fragmented can be inefficient), I needed a way to "unmark" if something was processed. For example, if I had consumed an item but needed to re-consume it, I needed a way to unmark that it was consumed. It also needed to maintain its order in the list. If I used a linked list I'd lose that ordering.

Anyways, the final use case looked something like this. The generator doesn't care what the order is, and when someone seeks they just update the internal head pointer.

[csharp]  
public void GenerateImages()  
{  
 OrderedConsumable = new OrderedConsumable\<int\>(SecondsToProcessList));

foreach (var second in OrderedConsumable)  
 {  
 MakeImageForSecond(second);  
 }  
}

public void Seek(int targetSeconds)  
{  
 OrderedConsumable.SetAsHead(seconds =\> targetSeconds == seconds);  
}  
[/csharp]

The nice thing is here, if I try to iterate over the consumable again, if everything is consumed nothing will return. This ensures that I don't reprocess anything that was already processed.

