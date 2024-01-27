---
layout: post
title: Byte arrays, typed values, binary reader, and fwrite
date: 2013-05-27 08:00:45.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- endianess
- x86
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561494804;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1268;}i:1;a:1:{s:2:"id";i:4286;}i:2;a:1:{s:2:"id";i:4213;}}}}

permalink: "/2013/05/27/byte-arrays-values-binary-reader-fwrite/"
---
I was trying to read a binary file created from a native app using the C# BinaryReader class but kept getting weird numbers. When I checked the hex in visual studio I saw that the bytes were backwards from what I expected, indicating endianess issues. This threw me for a loop since I was writing the file from C++ on the same machine that I was reading the file in C# in. Also, I wasn't sending any data over the network so I was a little confused. Endianess is usually an issue across machine architectures or over the network.

The issue is that I ran into an endianess problem when writing values byte by byte, versus by using the actual data type of an object. Let me demonstrate the issue

What happens if I write 65297 (0xFF11) using C++

```cpp
  
#include "stdafx.h"  
#include "fstream"

int \_tmain(int argc, \_TCHAR\* argv[])  
{  
 char buffer[] = { 0xFF, 0x11 };

auto \_stream = fopen("test2.out","wb");

fwrite(buffer, 1, sizeof(buffer), \_stream);

fclose(\_stream);  
}  

```

And read it in using the following C# code

```csharp
  
public void ReadBinary()  
{  
 using (var reader = new BinaryReader(new FileStream(@"test2.out", FileMode.Open)))  
 {  
 // read two bytes and print them out in hex  
 foreach (var b in reader.ReadBytes(2))  
 {  
 Console.Write("{0:X}", b);  
 }

Console.WriteLine();

// go back to the beginning  
 reader.BaseStream.Seek(0, SeekOrigin.Begin);

// read a two byte short and print it out in hex  
 var val = reader.ReadUInt16();

Console.WriteLine("{0:X}", val);  
 }  
}  

```

What would you expect I get? You might think we get the same thing both times, a 16 bit unsigned integer (2 bytes) and reading two bytes from the file should be the same right?

Actually, I got

```
  
FF11 \<-- reading in two bytes  
11FF \<-- reading in a two byte short  

```

What gives?

Turns out that since I'm on a little endian system (intel x86), when you read data as a typed structure it will always read little endian. The binary reader class in C# reads little endian, and fwrite in C++ will write little endian, as long as you aren't writing a value byte by byte.

When you write a value byte by byte it doesn't go through the correct endianess conversion. This means that you should make sure to use consistent write semantics. If you are going to write values byte by byte always write them byte by byte. If you are going to use typed data, always write with typed data. If you mix the write paradigms you can get into weird situations where some numbers are "big endian" (by writing it byte by byte), and some other values are little endian (by using typed data).

Here's a good quote from the ibm blog on [writing endianness independent code](http://www.ibm.com/developerworks/aix/library/au-endianc/?ca=drs-) summarizing the effect:

> Endianness does matter when you use a type cast that depends on a certain endian being in use.

If you do happen to need to write byte by byte, and you want to read values in directly as casted types in C#, you can make use of Jon Skeet's [MiscUtil](http://www.yoda.arachsys.com/csharp/miscutil/) which contains a big endian and little endian binary reader/writer class. By using the big endian reader you can now read files where you wrote them from C++ byte by byte.

Here is a fixed version

```csharp
  
using (var reader = new EndianBinaryReader(new BigEndianBitConverter(), new FileStream(@test2.out", FileMode.Open)))  
{  
 foreach (var b in reader.ReadBytes(2))  
 {  
 Console.Write("{0:X}", b);  
 }

Console.WriteLine();

reader.BaseStream.Seek(0, SeekOrigin.Begin);

var val = reader.ReadUInt16();

Console.WriteLine("{0:X}", val);

}  

```

Which spits out

```
  
FF11  
FF11  

```

