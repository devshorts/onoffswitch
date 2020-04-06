---
layout: post
title: Reading socket commands
date: 2013-10-29 17:00:43.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- active patterns
- F#
- Sockets
- TCP
meta:
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561407950;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1268;}i:1;a:1:{s:2:"id";i:1587;}i:2;a:1:{s:2:"id";i:4737;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/10/29/reading-socket-commands/"
---
A few weeks ago I was working on a sample application that would simulate a complex state machine. The idea is that there is one control room, and many slave rooms, where each slave room has its own state. The control room can dispatch a state advance or state reverse to any room or collection of rooms, as well as query room states, and other room metadata.

But to do this I need a way to get commands from the control room in order to know what to do. In my application clients were connected via tcp sockets and I wanted commands to be newline seperated. This made it easy to test out via a local telnet (I didn't need to design any binary protocol).

## The socket

You can never assume you've read what you want off a socket, since you're only ever guaranteed 1 or more bytes when a read succeeds. This means you need to continue to read until you've read however much you expected.

[fsharp]  
/// Listens on a tcp client and returns a seq\<byte[]\> of all  
/// found data  
let rec private listenOnClient (client:TcpClient) =  
 seq {  
 let stream = client.GetStream()

let bytes = Array.create 4096 (byte 0)  
 let read = stream.Read(bytes, 0, 4096)  
 if read \> 0 then  
 yield bytes.[0..read - 1]  
 yield! listenOnClient client  
 }

[/fsharp]

This function yields a seq of byte arrays each time the socket succeeds in a read. I'm reading only up to a 4096 buffer and leveraging F# array slicing to return the bytes that were actually read. After a read, the function calls itself and continues to yield byte arrays forever.

## Converting byte arrays to strings

The next step is taking those byte arrays and creating statements out of them. This means piecing them together and determining where newlines are. For example, if you read packets like

[code]  
Th  
is is a comm  
an  
d\n  
[/code]

It should really be handled like

[code]  
This is a command\n  
[/code]

To do this, I first map the bytes to utf8 strings, and use a string builder to aggregate lines. By using the string split function, I can tell (by empty entries) where newlines appeared, and whether or not a final terminating newline exists. For any statements that are terminated by a newline I can yield the entire command.

[fsharp]  
/// Reads off the client socket and aggregates commands that are seperated by newlines  
let packets (client:TcpClient) : seq\<string\> =  
 let filterEmpty = Seq.filter ((\<\>) String.Empty)  
 seq {  
 let builder = new StringBuilder()  
 for str in client |\> listenOnClient |\> Seq.map System.Text.ASCIIEncoding.UTF8.GetString do

let wordsWithBlanks = (builder.ToString() + str).Split([|'\r'; '\n'|])

builder.Clear() |\> ignore

// this means we got a newline following the last string so we have a  
 // group of totally valid commands  
 if Seq.last wordsWithBlanks = String.Empty then  
 for entry in wordsWithBlanks |\> filterEmpty do yield entry  
 else  
 // we didn't get a complete final command, so process all the other ones  
 let nonEmpties = wordsWithBlanks |\> filterEmpty

builder.Append (Seq.last nonEmpties) |\> ignore

for entry in (Seq.take (Seq.length nonEmpties - 1) nonEmpties) do  
 yield entry  
 }  
[/fsharp]

## Listening for commands

Now it's easy to leverage this function

[fsharp highlight="8"]  
let rec private listenForControlCommands (agentRepo:AgentRepo) client =  
 async {  
 let postFlip mailbox msg = post msg mailbox  
 let postToControl = postFlip agentRepo.Control

do! Async.SwitchToNewThread()  
 try  
 for message in client |\> packets do  
 match message with  
 | AdvanceCmd roomNum -\> postToControl \<| ControlInterfaceMsg.Advance roomNum  
 | ReverseCmd roomNum -\> postToControl \<| ControlInterfaceMsg.Reverse roomNum  
 | StartPreview roomNum -\> postToControl \<| ControlInterfaceMsg.StartPreview roomNum  
 | StartStreaming roomNum -\> postToControl \<| ControlInterfaceMsg.StartStreaming roomNum  
 | Record roomNum -\> postToControl \<| ControlInterfaceMsg.Record roomNum  
 | ResetRoom roomNum -\> postToControl \<| ControlInterfaceMsg.Reset roomNum  
 | QueryRoom roomNum -\> do! agentRepo |\> queryRoom roomNum client  
 | \_ -\> postToControl \<| ControlInterfaceMsg.Broadcast ("Unknown control sequence " + message)  
 with  
 | exn -\> postToControl (ControlInterfaceMsg.Disconnect client)  
 }  
[/fsharp]

Where the messages are matched with active patterns that parse the strings such as

[fsharp]  
let (|AdvanceCmd|\_|) (str:string) =  
 if str.StartsWith("advance ") then  
 str.Replace("advance ","").Trim() |\> Convert.ToInt32 |\> Some  
 else None  
[/fsharp]

The great thing about this is you hide all the string handling and deal only with strongly typed, high level patterns. Adding new commands is just a matter of creating a new active pattern and updating the message match in the `listenForControlCommands` function.

