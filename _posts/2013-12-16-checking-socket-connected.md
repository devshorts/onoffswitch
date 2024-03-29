---
layout: post
title: Checking if a socket is connected
date: 2013-12-16 08:00:29.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- F#
- Sockets
- TCP
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561771227;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4286;}i:1;a:1:{s:2:"id";i:1268;}i:2;a:1:{s:2:"id";i:1587;}}}}

permalink: "/2013/12/16/checking-socket-connected/"
---
Testing if a socket is still open isn't as easy at it sounds. Anyone who has ever dealt with socket programming knows this is hassle. The general pattern is to poll on the socket to see if its still available, usually by sitting in an infinite loop. However, with f# this can be done more elegantly using async and some decoupled functions.

First lets write an async function that monitors a socket and returns true if its connected or false if its not. The polling interval is set to 10 microseconds. This is because if you set the interval to a negative integer (representing to wait indefinitely for a response), it won't return until data is written to the socket. If, however, you have a very short poll time, you will be able to detect when the socket is closed without having to write data to it.

```fsharp
  
/// Async worker to say whether a socket is connected or not  
let isConnected (client:TcpClient) =  
 async {  
 return  
 try  
 if client.Client.Poll(10, SelectMode.SelectWrite) && not \<| client.Client.Poll(10, SelectMode.SelectError) then  
 let checkConn = Array.create 1 (byte 0)  
 if client.Client.Receive(checkConn, SocketFlags.Peek) = 0 then  
 false  
 else  
 true  
 else  
 false  
 with  
 | exn -\> false  
 }  

```

Next we can create a simple monitor function to check on a predicate, and if the predicate returns false (i.e not connected) it'll execute an action. Otherwise it'll call itself and monitor the socket again. This is important since the poll will exit once it determines that the socket is connected or not.

```fsharp
  
let rec monitor predicate onAction client =  
 async {  
 let! isConnected = predicate client  
 if not isConnected then  
 onAction client  
 else  
 return! monitor predicate onAction client  
 }  

```

Then, to use the monitor function all you have to do is something like this

```fsharp
  
monitor isConnected onDisconnect client |\> Async.Start  

```

Where `monitor` is the generic async monitor function, `isConnected` is the async socket poll, `onDisconnect` is a function to call when a client disconnects, and `client` is a tcpClient socket connection. The great thing about this is that you don't need to have a seperate thread for each open connection to act as a monitor/handler.

