---
layout: post
title: Cassandra DB migrations
date: 2016-01-23 23:03:20.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- cassandra
- migration
- paradoxical
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554980380;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4750;}i:1;a:1:{s:2:"id";i:4784;}i:2;a:1:{s:2:"id";i:3452;}}}}

permalink: "/2016/01/23/cassandra-db-migrations/"
---
When doing any application that involves a persistent data storage you usually need a way to upgrade and change your database using a set of scripts. Working with patterns like ActiveRecord you get easy up/down by version migrations. But with cassandra, which traditionally was schemaless, there aren't that many tools out there to do this.

One thing we have been using at my work and at paradoxical is a simple java based [cassandra loader tool](https://github.com/paradoxical-io/cassandra-loader) that does "up" migrations based on db version scripts.

Assuming you have a folder in your application that stores db scripts like

```
  
db/scripts/01\_init.cql  
db/scripts/02\_add\_thing.cql  
..  
db/sripts/10\_migrate\_users.cql  
..  

```

Then each script corresponds to a particular db version state. It's current state depends on all previous states. Our cassandra loader tracks db versions in a `db_version` table and lets you apply runners against a keyspace to move your schema (and data) to the target version. If your db is already at a version it does nothing, or if your db is a few versions back the runner will only run the required versions to get you to latest (or to the version number you want).

Taking this one step further, when working at least in Java we have the luxury of using [cassandra-unit](https://github.com/jsevellec/cassandra-unit) to actually run an embedded cassandra instance available for unit or integration tests. This way you don't need to mock out your database, you actually run all your db calls through the embedded cassandra. We use this heavily in [cassieq](https://github.com/paradoxical-io/cassieq) (a distributed queue based on cassandra).

One thing our cassandra loader can do is be run in library mode, where you give it the same set of db scripts and you can build a fresh db for your integration tests:

```java
  
public static Session create() throws Exception {  
 return CqlUnitDb.create("../db/scripts");  
}  

```

Running the loader in standalone mode (by downloading the `runner` [maven classifier](https://repo1.maven.org/maven2/io/paradoxical/cassandra.loader/1.1)) lets you run the migration runner in your console:

```
  
\> java -jar cassandra.loader-runner.jar

Unexpected exception:Missing required options: ip, u, pw, k  
usage: Main  
 -f,--file-path \<arg\> CQL File Path (default =  
 ../db/src/main/resources)  
 -ip \<arg\> Cassandra IP Address  
 -k,--keyspace \<arg\> Cassandra Keyspace  
 -p,--port \<arg\> Cassandra Port (default = 9042)  
 -pw,--password \<arg\> Cassandra Password  
 -recreateDatabase Deletes all tables. WARNING all  
 data will be deleted!  
 -u,--username \<arg\> Cassandra Username  
 -v,--upgrade-version \<arg\> Upgrade to Version  

```

The advantage to unifying all of this is that you can test your db scripts in isolation and be confident that they work!

