---
layout: post
title: Dalloc - coordinating resource distribution using hazelcast
date: 2016-01-22 01:54:44.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dalloc
- distributed
- hazelcast
- paradoxical
- paxos
- quorum
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554534280;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4839;}i:1;a:1:{s:2:"id";i:4750;}i:2;a:1:{s:2:"id";i:4800;}}}}

permalink: "/2016/01/22/coordinating-resource-distribution-hazelcast/"
---
A fun problem that has come up during the implementation of [cassieq](https://github.com/paradoxical-io/cassieq) (a distributed queue based on cassandra) is how to evenly distribute resources across a group of machines. There is a scenario in cassieq where writes can be delayed, and as such there is a custom worker in the app (by queue) who watches a queue to see if a delayed write comes in and republishes the message to a bucket later on. It's transparent to the user, but if we have multiple workers on the same queue we could potentially republish the message twice. While technically that falls within the SLA we've set for cassieq (at least once delivery) it'd be nice to avoid this particular race condition.

To solve this, I've clustered the cassieq instances together using [hazelcast](http://hazelcast.org/). Hazelcast is a pretty cool library since it abstracts away member discovery/connection and gives you events on membership changes to make it easy for you to build distributed data grids. It also has a lot of great primitives that are useful in building distributed workflows. Using hazelcast, I've built a simple resource distributor that uses shared distributed locks and a master set of allocations across cluster members to coordinate who can "grab" which resource.

For the impatient you can get [dalloc](https://github.com/paradoxical-io/dalloc) from

[java]  
\<dependency\>  
 \<groupId\>io.paradoxical\</groupId\>  
 \<artifactId\>dalloc\</artifactId\>  
 \<version\>1.0\</version\>  
\</dependency\>  
[/java]

The general idea in dalloc is that each node creates a resource allocator who is bound to a resource group name (like "Queues"). Each node supplies a function to the allocator that generates the master set of resources to use, and a callback for when resources are allocated. The callback is so you can wire in async events and when allocations need to be rebalanced outside of a manual invocation (like cluster member/join).

The entire resource allocation library API deals with abstractions on what a resource is, and lets the client map their internal resource into a `ResourceIdentity`. For cassieq, it's a queue id.

When an allocation is triggered (either manually or via a member join/leave event) the following occurs:

- Try and acquire a shared lock for a finite period of time
- If you acquired the lock, acquire a map of what has been allocated to everyone else and compare what is available from your master set to what is available
- Given the size of the current cluster, determine how many resources you are allowed to claim (by even distribution). If you don't have your entire set claimed, take as many as you can to fill up. If you have too many claimed, give some resources up
- Persist your changes to the master state map
- Dispatch to your callback what the new set of resources should be

Hazelcast supports distributed maps, where part of the map is sharded by its map key on different nodes. However, I'm actually explicitly NOT distributing the map across the cluster. I've put ownership of the resource set on "one" node (but the map is replicated so if that node goes down the map still exists). This is because each node is going to have to try and do a claim. If each node claims, and then calls to every other node, thats n^2 IO operations. Compare that to every node making N operations.

The library also supports bypassing this mechanism and instead supports a much more "low-tech" solution of manual allocation. All this means is that you pre-define how many nodes there should be, and which node number a node is. Then each node sorts the input data and grabs a specific slice out of the input set based on its id. It doesn't give any guarantees to non-overlap, but it does give you an 80% solution to a hard problem.

[Jake](https://twitter.com/jakeswenson), the other [paradoxical](http://paradoxical.io/) member suggested that there could be a nice alternative solution using a similar broadcast style of quorum using paxos. Each node broadcasts what it's claiming and the nodes agree on who is allowed to do what. I probably wouldn't use hazelcast for that, though the primitives of paxos (talking to all members of a cluster) are there and it'd be interesting to build paxos on top of hazelcast now that I think about it...

Anyways, abstracting distributed resource allocation is nice, because as we make improvements to how we want to tune the allocation algorithms all dependent services get it for free. And free stuff is my favorite.

