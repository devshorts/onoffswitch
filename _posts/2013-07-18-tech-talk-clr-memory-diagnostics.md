---
layout: post
title: 'Tech talk:  CLR Memory Diagnostics'
date: 2013-07-18 15:35:36.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- clr
- Debugging
- heaps
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559337772;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4596;}i:1;a:1:{s:2:"id";i:4463;}i:2;a:1:{s:2:"id";i:3497;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/07/18/tech-talk-clr-memory-diagnostics/"
---
Today's tech talk we discussed the recent release from Microsoft of [ClrMD](http://blogs.msdn.com/b/dotnet/archive/2013/05/01/net-crash-dump-and-live-process-inspection.aspx) that lets you attach and debug processes using an exposed API. You used to be able to do this in WinDbg using the SOS plugin, but now they've wrapped SOS in a managed dll that you can use to inspect CLR process information. The nice thing about this is you can now automate debugging inspections. It's now as easy as

[csharp]  
int pid = Process.GetProcessesByName("TestApplication")[0].Id;

using (DataTarget dataTarget = DataTarget.AttachToProcess(pid, 5000))  
{  
 string dacLocation = dataTarget.ClrVersions[0].TryGetDacLocation();  
 ClrRuntime runtime = dataTarget.CreateRuntime(dacLocation);

ClrHeap heap = runtime.GetHeap();

foreach (ulong obj in heap.EnumerateObjects())  
 {  
 ClrType type = heap.GetObjectType(obj);  
 ulong size = type.GetSize(obj);  
 Console.WriteLine("{0,12:X} {1,8:n0} {2}", obj, size, type.Name);  
 }  
}  
[/csharp]

ClrMD lets you take stack snapshots of running threads, iterate through all objects in the heap and get their values out, show all loaded modules and more. If you combine it with [ScriptCS](http://scriptcs.net/) you've got a really powerful debugging tool. What I liked about ClrMD is that you can use the same API to attach to running processes (in modes where you can pause the app while running, or run without pausing the attached app) as you can with process dumps.

While it is nice to be able to inspect the heap and stacks, I found that it's not totally trivial to get object information. Since all your queries return pointers to values on the heap (unless its a primitive object), you need to recursively go through the entire object reference to find all the details. If you have the source, symbols, and a crash dump I think it's easier to just toss it into visual studio to get this information. Still, if you don't have access to this and need to investigate or automate error detection in a low level fashion, then this is an amazing tool.

For more reading check out

[https://nuget.org/packages/Microsoft.Diagnostics.Runtime](https://nuget.org/packages/Microsoft.Diagnostics.Runtime)

[http://www.infoq.com/news/2013/06/microsoft-diagnostics-runtime](http://www.infoq.com/news/2013/06/microsoft-diagnostics-runtime)

[http://www.piotrwalat.net/clr-diagnostics-with-clrmd-and-scriptcs-repl-scriptcs-clrdiagnostics/](http://www.piotrwalat.net/clr-diagnostics-with-clrmd-and-scriptcs-repl-scriptcs-clrdiagnostics/)

[http://www.programmingtidbits.com/post/2013/05/09/CLR-Memory-Diagnostics-Released.aspx](http://www.programmingtidbits.com/post/2013/05/09/CLR-Memory-Diagnostics-Released.aspx)

