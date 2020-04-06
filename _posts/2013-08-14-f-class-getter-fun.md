---
layout: post
title: F# class getter fun
date: 2013-08-14 16:21:49.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- c#
- classes
- F#
- initialization
- neo4j
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558960558;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4463;}i:1;a:1:{s:2:"id";i:4244;}i:2;a:1:{s:2:"id";i:4028;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/08/14/f-class-getter-fun/"
---
I was playing with Neo4J (following a recent post I stumbled upon by [Sergey Tihon](http://sergeytihon.wordpress.com/2013/03/27/using-neo4j-graph-db-with-f/)), and had everything wired up and ready to test out, but when I tried running my code I kept getting errors saying that I hadn't connected to the neo4j database. This puzzled me because I had clearly called connect, but every time I tried to access my connection object I got an error.

The issue was that I didn't realize that f# class members are always deferred. It makes sense that they are after I traced through it, but I couldn't spot the bug for the life of me at first.

My code looked like this:

[fsharp]  
module Connection =  
 type Connection (dbUrl) =

member x.client = new GraphClient(new Uri(dbUrl))

member x.create item = x.client.Create item

member x.connect() =  
 x.client.Connect()  
 x  
[/fsharp]

If I had more experience with F# I probably would have spotted this right away, but it took me a while to figure out what was going on. The issue here is

[fsharp highlight="4"]  
module Connection =  
 type Connection (dbUrl) =

member x.client = new GraphClient(new Uri(dbUrl))

member x.create item = x.client.Create item

member x.connect() =  
 x.client.Connect()  
 x  
[/fsharp]

Which compiles into

[csharp highlight="15"]  
 [AutoOpen]  
 [CompilationMapping(SourceConstructFlags.Module)]  
 public static class Connection  
 {  
 [CompilationMapping(SourceConstructFlags.ObjectType)]  
 [Serializable]  
 public class Connection  
 {  
 internal string dbUrl;

public GraphClient client  
 {  
 get  
 {  
 return new GraphClient(new Uri(this.dbUrl));  
 }  
 }

public Connection(string dbUrl)  
 {  
 Connection.Connection connection = this;  
 this.dbUrl = dbUrl;  
 }

public NodeReference\<a\> create\<a\>(a item) where a : class  
 {  
 return GraphClientExtensions.Create\<a\>((IGraphClient) this.client, item, new IRelationshipAllowingParticipantNode\<a\>[0]);  
 }

public Connection.Connection connect()  
 {  
 this.client.Connect();  
 return this;  
 }  
 }  
 }  
[/csharp]

Clear as day now. Each time you call the property it returns a new instance. I had assumed that since the member wasn't a function that it would be a property, not an auto wrapped getter.

The fix was easy:

[fsharp]  
module Connection =  
 type Connection (dbUrl) =

let graphConnection = new GraphClient(new Uri(dbUrl))

member x.client = graphConnection

member x.create item = x.client.Create item

member x.connect() =  
 x.client.Connect()  
 x  
[/fsharp]

Which now generates

[csharp]  
 [AutoOpen]  
 [CompilationMapping(SourceConstructFlags.Module)]  
 public static class Connection  
 {  
 [CompilationMapping(SourceConstructFlags.ObjectType)]  
 [Serializable]  
 public class Connection  
 {  
 internal GraphClient graphConnection;

public GraphClient client  
 {  
 get  
 {  
 return this.graphConnection;  
 }  
 }

public Connection(string dbUrl)  
 {  
 Connection.Connection connection = this;  
 this.graphConnection = new GraphClient(new Uri(dbUrl));  
 }

public NodeReference\<a\> create\<a\>(a item) where a : class  
 {  
 return GraphClientExtensions.Create\<a\>((IGraphClient) this.client, item, new IRelationshipAllowingParticipantNode\<a\>[0]);  
 }

public Connection.Connection connect()  
 {  
 this.client.Connect();  
 return this;  
 }  
 }  
 }  
[/csharp]

That's more like it

