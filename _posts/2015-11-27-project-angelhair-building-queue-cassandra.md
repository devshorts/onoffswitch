---
layout: post
title: 'Project angelhair: Building a queue on cassandra'
date: 2015-11-27 22:35:44.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- cassandra
- docker
- java
- queue
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559648543;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4839;}i:1;a:1:{s:2:"id";i:4800;}i:2;a:1:{s:2:"id";i:4783;}}}}

permalink: "/2015/11/27/project-angelhair-building-queue-cassandra/"
---


Edit: this project has since been moved to CassieQ: https://github.com/paradoxical-io/cassieq

A few weeks ago my work had a hack day and I got together with some of my coworker friends and we decided to build a queue on top of Cassandra.

For the impatient, give it a try (docker hub):

```
docker run -it \
    -e CLUSTER_NAME="" \
    -e KEYSPACE="" \
    -e CONTACT_POINTS="" \
    -e USERNAME="" \
    -e PASSWORD="" \
    -e USE_SSL="" \
    -e DATA_CENTER="" \
    -e METRICS_GRAPHITE "true" \
    -e GRAPHITE_PREFIX=" \
    -e GRAPHITE_URL=""  \
    onoffswitch/angelhair

```

The core features for what we called Project Angelhair was to handle:

- long term events (so many events that AMQ or RMQ might run out of storage space)

- connectionless - wanted to use http

- invisibility - need messages to disappear when they are processing but be able to come back

- highly scaleable - wanted to distribute a docker container that just did all the work

Building a queue on cassandra isn't a trivial task and is rife with problems.  In fact, this is pretty well known and in general the consensus is don't build a queue on Cassandra.

But why not?  There are a few reasons.  In general, the question you want to answer with a queue is "what haven't I seen". A simple way to do this is when a message is consumed to delete it.  However, with cassandra, deletes aren't immediate. They are tombstoned, so they will exist for the compaction period. This means even if you have only 1 message in your queue, cassandra has to scan all the old deleted messages before it finds it. With high load this can be a LOT of extra work. But thats not the only problem.  You have problems of how to distribute your messages across the ring. If you put all your messages for a queue into one partition key now you haven't evenly distributed your messages and have a skewed distribution of work. This is going to manifest in really poor performance.

On top of all of that, cassandra has poor support for atomic transactions, so you can't easily say "let me get, process, and consume" in one atomic action.  Backing stores that are owned by a master (like sqlserver) let you do atomic actions much better since they have either have an elected leader who can manage this or are a single box. Cassandra isn't so lucky.

Given all the problems described, it may seem insane to build a queue on Cassandra. But cassandra is a great datastore that is massively horizontally scaleable. It also exists at a lot of organizations already. Being able to use a horizontally scaleable data store means you can ingest incredible amounts of messages.

How does angelhair work?

Angelhair works with 3 pointers into a queue.

A reader bucket pointer

A repair bucket pointer

An invisibility pointer

In order to scale and efficiently act as a queue we need to leverage cassandra partitioning capabilities. Queues are actually messages bucketized into a fixed size group called a bucket. Each message is assigned a monotonically increasing id that maps itself into a bucket.  For example, if the bucket is size 20 and you have id 21, that maps into bucket 1 (21/20). This is done using a table in cassandra whose only job is to provide monotonic values for a queue:

```
CREATE TABLE monoton (
  queuename text PRIMARY KEY,
  value bigint
);
```

By bucketizing messages we can distribute messages across the cassandra clusters.

Messages are always put into the bucket they correlate to, regardless if previous buckets are full.  This means that messages just keep getting put into the end, as fast as possible.

Given that messages are put into their corresponding bucket, the reader has a pointer to its active bucket (the reader bucket pointer) and scans the bucket for unacked visible messages. If the bucket is full it tombstones the bucket indicating that the bucket is closed for processing. If the bucket is NOT full, but all messages in the bucket are consumed (or being processed) AND the monotonic pointer has already advanced to the next bucket, the current bucket is also tombstoned. This means no more messages will ever show up in the current bucket... sort of

Repairing delayed writes

Without synchronizing reads and writes you can run into a situation  where you can have a delayed write. For example, assume you generate monotonic ids in this sequence:

```
Id 19
Id 20
Write 20 <-- bucket advances to bucket 1
             (assuming bucket size of 20) and
             bucket 0 is tombstoned (closed)
Write 19 <-- but message 19 writes into
             bucket 0, even though 0
             was tombstoned!
```

In this scenario id 20 advances the monotonic bucket to bucket 1 (given buckets are size 20). That means the reader tombstones bucket 0. But what happens to message 19? We don't want to lose it, but as far as the reader is concerned it's moved onto bucket 1 and off of bucket 0.

This is where the concept of a repair worker comes into play. The repair worker's job is to slowly follow the reader and wait for tombstoned buckets. It has its own pointer (the repair bucket pointer) and polls to find when a bucket is tombstoned.  When a bucket is tombstoned the repair worker will wait for a configured timeout for out of order missing messages to appear. This means if a slightly delayed write occurs then the repair worker will actually pick it up and then republish it to the last active bucket.  We're gambling on probability here, the assumption is that if a message is going to be successfully written then it will be written within time T. That time is configurable when you create the queue.

But there is also a scenario like this:

```
Id 19
Id 20
!!Write 19 ---
\> This actually dies and fails to write!  
Write 20
```

In this scenario we claimed Id's 19 and 20, but 19 failed to write. Once 20 is consumed the reader tombstones the bucket and the repair worker kicks in. But 19 isn't ever going to show up! In this case, the repair worker waits for the configured time and if after that time the message _isn't_ written then we assume that that message is dead and will never be processed. Then the repair worker advances its pointer and moves on.

This means we don't necessarily guarantee FIFO, however we do (reasonably) guarantee messages will appear. The repair worker never moves past a non completed bucket, though since its just a pointer we can always repair the repair worker by moving the pointer back.

# Invisibility

Now the question comes up as how to deal with invisibility of messages. Invisible messages are important since with a conncectionless protocol (like http) we need to know if a message worker is dead and its message has to go back for processing. In queues like RMQ this is detected when a channel is disconnected (i.e. the connection is lost). With http not so lucky.

To track invisibility there is a separate pointer tracking the last invisible pointer. When a read comes in, we first check the invisibility pointer to see if that message is now visible.

If it is, we can return it. If not, get the next available message.

If the current invisible pointer is already acked then we need to find the next invisible pointer. This next invisible pointer is the first non-acked non-visible message. If there isn't one in the current bucket, the invisibility pointer moves to the next bucket until it finds one or no messages exist, but never move past a message that hasn't been delivered before. This way it won't accidentally skip a message that hasn't been sent out yet.

If however, there are two messages that get picked up at the same time the invis pointer is scanning through the invis pointer could choose the wrong id. In order to prevent this, we update the invis pointer to the destination if it's less than the current (i.e. we need to move back), or if its not then only update if the current reader owns the current invis pointer (doing an atomic update).

# API

Angelhair has a simple API.

- Put a message into a queue (and optionally specify an initial invisiblity)  
- Get a message from a queue  
- Ack the message using the message pop reciept (which is an encoded version and id metadata). The pop reciept is unique for each message dequeue. If a message comes back alive and is available for processing again it gets a new pop recipet. This also lets us identify a unique consumer of a message since the current atomic version of the message is encoded in the pop reciept.

Doesn't get much easier than that!

# Conclusion

There are a couple implementations of queues on cassandra out there that we found while researching this. One is from [netflix](https://github.com/Netflix/astyanax/wiki/Message-Queue) but their implementation builds a lock system on top of cassandra and coordinates reads/writes using locking. Some other implementations used wide rows (or CQL lists in a single row) to get around the tombstoning, but that limits the number of messages in your "queue" to 64k messages.

While we haven't tested angelhair in a stressed environment, we've decided to give it a go in some non critical areas in our internal tooling. But so far we've had great success with it!

