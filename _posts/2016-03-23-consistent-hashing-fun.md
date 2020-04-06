---
layout: post
title: Consistent hashing for fun
date: 2016-03-23 23:01:20.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- distributed
- scala
- toy
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554617742;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4805;}i:1;a:1:{s:2:"id";i:4783;}i:2;a:1:{s:2:"id";i:4699;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/03/23/consistent-hashing-fun/"
---
I think [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) is pretty fascinating. It lets you define a ring of machines that shard out data by a hash value. Imagine that your hash space is 0 -\> Int.Max, and you have 2 machines. Well one machine gets all values hashed from 0 -\> Int.Max/2 and the other from Int.Max/2 -\> Int.Max. Clever. This is one of the major algorithms of distributed systems like cassandra and dynamoDB.

For a good visualization, check out this [blog post](http://www.paperplanes.de/2011/12/9/the-magic-of-consistent-hashing.html/).

The fun stuff happens when you want to add replication and fault tolerance to your hashing. Now you need to have replicants and manage when machines join and add. When someone joins, you need to re-partition the space evenly and re-distribute the values that were previously held.

Something similar when you have a node leave, you need to make sure that whatever it was responsible for in its primray space AND the things it was responsible for as a secondary replicant, are re-redistributed amongst the remaining nodes.

But the beauty of consistent hashing is that the replication basically happens for free! And so does redistribution!

Since my new feature is in all in Scala, I figured I'd write something up to see how this might play out in scala.

For the impatient, the full source is[here](https://github.com/devshorts/consistent-hasher/blob/better/src/main/scala/com/example).

First I started with some data types

[scala]  
case class HashValue(value: String) extends AnyRef

case class HashKey(key: Int) extends AnyRef with Ordered[HashKey] {  
 override def compare(that: HashKey): Int = key.compare(that.key)  
}

object HashKey {  
 def safe(key: Int) = new HashKey(Math.abs(key))  
}

case class HashRange(minHash: HashKey, maxHash: HashKey) extends Ordered[HashRange] {  
 override def compare(that: HashRange): Int = minHash.compare(that.minHash)  
}  
[/scala]

I chose to wrap the key in a positive space since it made things slightly easier. In reality you want to use md5 or some actual hashing function, but I relied on the hash code here.

And then a machine to hold values:

[scala]  
import scala.collection.immutable.TreeMap

class Machine[TValue](val id: String) {  
 private var map: TreeMap[HashKey, TValue] = new TreeMap[HashKey, TValue]()

def add(key: HashKey, value: TValue): Unit = {  
 map = map + (key -\> value)  
 }

def get(hashKey: HashKey): Option[TValue] = {  
 map.get(hashKey)  
 }

def getValuesInHashRange(hashRange: HashRange): Seq[(HashKey, TValue)] ={  
 map.range(hashRange.minHash, hashRange.maxHash).toSeq  
 }

def keepOnly(hashRanges: Seq[HashRange]): Seq[(HashKey, TValue)] = {  
 val keepOnly: TreeMap[HashKey, TValue] =  
 hashRanges  
 .map(range =\> map.range(range.minHash, range.maxHash))  
 .fold(map.empty) { (tree1, tree2) =\> tree1 ++ tree2 }

val dropped = map.filter { case (k, v) =\> !keepOnly.contains(k) }

map = keepOnly

dropped.toSeq  
 }  
}  
[/scala]

A machine keeps a sorted tree map of hash values. This lets me really quickly get things within ranges. For example, when we re-partition a machine, it's no longer responsible for the entire range set that it was before. But it may still be responsible for parts of it. So we want to be able to tell a machine _hey, keep ranges 0-5, 12-20, but drop everything else_. The tree map lets me do this really nicely.

Now for the fun part, the actual consistent hashing stuff.

Given a set of machines, we need to define how the circular partitions is defined

[scala]  
private def getPartitions(machines: Seq[Machine[TValue]]): Seq[(HashRange, Machine[TValue])] = {  
 val replicatedRanges: Seq[HashRange] = Stream.continually(defineRanges(machines.size)).flatten

val infiteMachines: Stream[Machine[TValue]] =  
 Stream.continually(machines.flatMap(List.fill(replicas)(\_))).flatten

replicatedRanges  
 .zip(infiteMachines)  
 .take(machines.size \* replicas)  
 .toList  
}  
[/scala]

What we want to make sure is that each node sits on multiple ranges, this gives us the replication factor. To do that I've duplicated the machines in the list by the replication factor, and made sure all the lists cycle around indefinteily, so while they are not evenly distributed around the ring (they are clustered) they do provide fault tolerance

Lets look at what it takes to put a value into the ring:

[scala]  
private def put(hashkey: HashKey, value: TValue): Unit = {  
 getReplicas(hashkey).foreach(\_.add(hashkey, value))  
}

private def getReplicas(hashKey: HashKey): Seq[Machine[TValue]] = {  
 partitions  
 .filter { case (range, machine) =\> hashKey \>= range.minHash && hashKey \< range.maxHash }  
 .map(\_.\_2)  
}  
[/scala]

We need to make sure that for each replica in the ring that sits on a hash range, that we insert it into that machine. Thats pretty easy, though we can improve this later with better lookups

Lets look at a get

[scala]  
def get(hashKey: TKey): Option[TValue] = {  
 val key = HashKey.safe(hashKey.hashCode())

getReplicas(key)  
 .map(\_.get(key))  
 .collectFirst { case Some(x) =\> x }  
}  
[/scala]

Also similar. Go through all the replicas, and find the first one to return a value

Now lets look how to add a machine into the ring

[scala]  
def addMachine(): Machine[TValue] = {  
 id += 1

val newMachine = new Machine[TValue]("machine-" + id)

val oldMachines = partitions.map(\_.\_2).distinct

partitions = getPartitions(Seq(newMachine) ++ oldMachines)

redistribute(partitions)

newMachine  
}  
[/scala]

So we first create a new list of machines, and then ask how to re-partition the ring. Then the keys in the ring need to redistribute themselves so that only the nodes who are responsible for certain ranges contain those keys

[scala]  
def redistribute(newPartitions: Seq[(HashRange, Machine[TValue])]) = {  
 newPartitions.groupBy { case (range, machine) =\> machine }  
 .flatMap { case (machine, ranges) =\> machine.keepOnly(ranges.map(\_.\_1)) }  
 .foreach { case (k, v) =\> put(k, v) }  
}  
[/scala]

Redistributing isn't that complicated either. We group all the nodes in the ring by the machine they are on, then for each machine we tell it to only keep values that are in its replicas. The machine `keepOnly` function takes a list of ranges and will remove and _return_ anything not in those ranges. We can now aggregate all the things that are "emitted" by the machines and re insert them into the right location

Removing a machine is really similiar

[scala]  
def removeMachine(machine: Machine[TValue]): Unit = {  
 val remainingMachines = partitions.filter { case (r, m) =\> !m.eq(machine) }.map(\_.\_2)

partitions = getPartitions(remainingMachines.distinct)

redistribute(partitions)  
}  
[/scala]

And thats all there is to it! Now we have a fast, simple consistent hasher.

