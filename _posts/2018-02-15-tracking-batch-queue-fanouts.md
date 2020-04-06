---
layout: post
title: Tracking batch queue fanouts
date: 2018-02-15 00:17:40.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- architecture
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"37550b67d263a3ce789993dc25046c5f";a:2:{s:7:"expires";i:1557076392;s:7:"payload";a:6:{i:0;a:1:{s:2:"id";i:4783;}i:1;a:1:{s:2:"id";i:4750;}i:2;a:1:{s:2:"id";i:4945;}i:3;a:1:{s:2:"id";i:1587;}i:4;a:1:{s:2:"id";i:532;}i:5;a:1:{s:2:"id";i:390;}}}}
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2018/02/15/tracking-batch-queue-fanouts/"
---
> Edit: This code now exists at [https://github.com/paradoxical-io/carlyle](https://github.com/paradoxical-io/carlyle)

When working in any resilient distributed system invariably queues come into play. You fire events to be handled into a queue, and you can horizontally scale workers out to churn through events.

One thing though that is difficult to do is to answer a question of _when is a batch of events finished?_ This is a common scenario when you have a single event causing a fan-out of other events. Imagine you have an event called `ProcessCatalog` and a catalog may have N catalog items. The first handler for `ProcessCatalog` may pull some data and fire N events for catalog items. You may want to know when all catalog items are processed by downstream listeners for the particular event.

It may seem easy though right? Just wait for the last item to come through. Ah, but distributed queues are loosely ordered. If an event fails and retries what used to be the last event is no longer the last.

What about having a counter somewhere? Well the same issue arises on event failure. If an event decrements the counter _twice_ (because it was retried) now the counter is no longer deterministic.

To do this thoroughly you need to track every individual item and ack it as its processed. That would work but it could be really costly. Imagine tracking 1B items some data store _for every batch_!

Lets pretend we go this route though, what else do we need to do? We need the concept of a batch, and items to track in the batch. If we have a workflow of opening a batch (creating a new one), adding items to it, then sealing the batch (no more items can be added to it) we can safely determine if the batch is empty. Without the concept of a `close/seal` you can have situations where you open a batch, fire off N events, N events are consumed ack'd by downstream systems, then you close the batch. When all N are ack'd by downstream systems they can't _know_ that there are no more incoming items since the batch is still open. Only when the batch is closed can you tell if the batch has been fully consumed. To that note, both the producer AND consumer need to check if the batch is closed. In the previous example, if all N items are ack'd before the batch is closed, when you go to close the batch it needs to return back that the batch is empty! In the other situation if the batch is closed, then all N items are ack'd the last item to be ack'd needs to return that the batch is empty.

Back to the problem of storing and transferring N batch item ID's though. What if instead of storing each item you leveraged a [bitfield](http://onoffswitch.net/bit-packing-pacman) representing a set of items? Now instead of N items you only need N bits to logically track every item. But you also may now need to send N logical ID's back to the client. We can also get around that by knowing that anytime you add a batch of items to a batch, for example, adding 1000 items to batch id 1, that this sub group batch can be inserted as a unique hash corresponding to a bitfield set to all 1's with 1000 bits (any extra bits set to 0 and ignored).

Returning to the client all you need to send back is the _hash_ and how many items are related to that hash. Now determistic id's can be created _client side_ that are of the form `batchId:hash:index`. When a client goes back to ack a message (or batch of messages) the ID contains enough information to

1. Locate all other hashes related to the batch
2. Get the bitfield for the ash
3. Flip the appropriate bit represented by the index

Something maybe like this

[scala]  
case class BatchItemGroupId(value: UUID) extends UuidValue

object BatchItemId {  
 def apply(batchId: BatchId, batchItemGroupId: BatchItemGroupId, index: Long): BatchItemId = BatchItemId(s"${batchId.value}:$batchItemGroupId:$index")

def generate(batchId: BatchId, batchItemGroupId: BatchItemGroupId, numberOfItems: Int): Iterable[BatchItemId] = {  
 Stream.from(0, step = 1).take(numberOfItems).map(idx =\> apply(batchId, batchItemGroupId, idx))  
 }

def generate(batchId: BatchId, batchItemGroupId: List[BatchItemGroupInsert]): Iterable[BatchItemId] = {  
 batchItemGroupId.toStream.flatMap(group =\> generate(batchId, group.id, group.upto))  
 }  
}

case class BatchItemId(value: String) extends StringValue {  
 val (batchId, batchItemGroupId, index) = value.split(':').toList match {  
 case batch::hash::index::\_ =\> (BatchId(batch.toLong), BatchItemGroupId(UUID.fromString(hash)), index.toLong)  
 case \_ =\> throw new RuntimeException(s"Invalid batch item id format $value")  
 }  
}

case class BatchId(value: Long) extends LongValue  
[/scala]

We also need an abstraction on top of a byte array that lets us toggle bits in it. It also lets you count how many bits are set. We'll need to know that so we can answer the question of "is this subgroup hash empty". I've shown bitmasking but I can show it again [at this gist](https://gist.github.com/devshorts/df36b7f042d8f64df382efb1a43c898a).

Now that we have that, all our sql needs to do is given a subgroup batch, pull the bitfield, put the bitfield into a `BitGroup`, flip the appropriate bits, write the bitfield back. Then it needs to query all batch groups for a batch, pull their blobs, and count their bits to determine if the batch is pending or complete.

If we're smart about it and limit the bitfield in MySQL to be large enough to get compression, but small enough to minimize over the wire overhead... say 1000 bytes =~ 10k (which works out to be about 8000 bits) we can store quit a bit with quite a little!

Unfortunately MySQL doesn't support bitwise operations on blobs (not till MySQL 8 apparently) so you need to pull the blob, manipulate it, then write it back. But you can safely do this in a transaction if you select the row using `FOR UPDATE` which provides row level read locking.

One last thing to think about before we call it a day though. There are all sorts of situations where batches can be opened but never closed. Imagine a client opens a batch, adds items to it, then dies. It may retry and create a new batch and add new items, but there is effectively an orphaned batch. To make our queue batcher work long term we need some asynchronous cleanup. You arbitrarily decide that any non-closed batches with no activity for N days get automatically deleted. Same with any closed-batches with no activity (maybe at a different interval). This lets the system deal with batches/events that are never going to complete.

Package this all up into a service with an API and blamo! Efficient queue batch tracking!

