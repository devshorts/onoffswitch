---
layout: post
title: Leadership election with cassandra
date: 2016-01-21 22:56:55.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- cassandra
- java
- leadership
- paradoxical
- quorum
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560857090;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4800;}i:1;a:1:{s:2:"id";i:4750;}i:2;a:1:{s:2:"id";i:4783;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/01/21/leadership-election-cassandra/"
---
Cassandra has a neat feature that lets you expire data in a column. Using this handy little feature, you can create simple leadership election using cassandra. The whole process is [described here](http://www.datastax.com/dev/blog/consensus-on-cassandra) which talks about leveraging Cassandras consensus and the column expiration to create leadership electors.

The idea is that a user will try and claim a slot for a period of time in a leadership table. If a slot is full, someone else has leadership. While the leader is still active they needs to heartbeat the table faster than the columns TTL to act as a keepalive. If it fails to heartbeat (i.e. it died) then its leadership claim can be relinquished and someone else can claim it. Unlike most leadership algorithms that claim a single "host" as a leader, I needed a way to create leaders sharded by some "group". I call this a "LeadershipGroup" and we can leverage the expiring columns in cassandra to do this!

To make this easier, I've wrapped this algorithm in a java library available [from paradoxical](https://github.com/paradoxical-io/cassandra.leadership). For the impatient

[java]  
\<dependency\>  
 \<groupId\>io.paradoxical\</groupId\>  
 \<artifactId\>cassandra-leadership\</artifactId\>  
 \<version\>1.0\</version\>  
\</dependency\>  
[/java]

The gist here is that you need to provide a schema similar to

[code]  
CREATE TABLE leadership\_election (  
 group text PRIMARY KEY,  
 leader\_id text  
);  
[/code]

Though the actual column names can be custom defined. You can define a leadership election factory using Guice like so

[java]  
public class LeadershipModule extends AbstractModule {  
 @Override  
 protected void configure() {  
 bind(LeadershipSchema.class).toInstance(LeadershipSchema.Default);

bind(LeadershipStatus.class).to(LeadershipStatusImpl.class);

bind(LeadershipElectionFactory.class).to(CassandraLeadershipElectionFactory.class);  
 }  
}  
[/java]

- `LeadershipStatus` is a class that lets you query who is leader for what "group". For example, you can have multiple workers competing for leadership of a certain resource. 
- `LeadershipSchema` is a class that defines what the column names in your schema are named. By default if you use the sample table above, the Default schema maps to that
- `LeadershipElectionFactory` is a class that gives you instances of LeadershipElection classes, and I've provided a cassandra leadership factory

Once we have a leader election we can try and claim leadership:

[java]  
final LeadershipElectionFactory factory = new CassandraLeadershipElectionFactory(session);

// create an election processor for a group id  
final LeadershipElection leadership = factory.create(LeadershipGroup.random());

final LeaderIdentity user1 = LeaderIdentity.valueOf("user1");

final LeaderIdentity user2 = LeaderIdentity.valueOf("user2");

assertThat(leadership.tryClaimLeader(user1, Duration.ofSeconds(2))).isPresent();

Thread.sleep(Duration.ofSeconds(3).toMillis());

assertThat(leadership.tryClaimLeader(user2, Duration.ofSeconds(3))).isPresent();  
[/java]

When you claim leadership you claim it for a period of time and if you get it you get a leadership token that you can heartbeat on. And now you have leadership!

As usual, full source available at my [github](https://github.com/paradoxical-io/cassandra.leadership)

