---
layout: post
title: 'Tech talk: Hacking droid'
date: 2013-04-04 20:20:37.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- droid
- java
meta:
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _edit_last: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560892211;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3285;}i:1;a:1:{s:2:"id";i:1268;}i:2;a:1:{s:2:"id";i:4699;}}}}

permalink: "/2013/04/04/tech-talk-hacking-droid/"
---
Todays tech talk was based off of a blog entry posted [by facebook](https://www.facebook.com/notes/facebook-engineering/under-the-hood-dalvik-patch-for-facebook-for-android/10151345597798920) recently where they described the things they needed to do to get their mobile app running on android OS Froyo (v 2.2).

The gist of the post was that facebook migrated a lot of javascript code to Java and then found themselves in an interesting situation: they were unable to load the app due to the number of methods declared being larger than the method metadata buffer could hold. The buffer, called "LinearAlloc", on this particular droid version was 5MB in size. Later versions were increased to 8MB which meant that the bug was no longer exhibited.

Facebook's engineers tried to break their application up into multiple droid executable files (dex), tried automatic source transformations to minimize their method calls, tried refactoring some of their code (but were unable to make siginifant headway due to the strong coupling they had), and ended up having to do some memory hacking insanity to find the source of the buffer and replace it with a pointer to another buffer that was larger. This way they could have enough space to store the methods they needed. They also put in a failsafe where they would scan the entire process memory heap looking for the correct buffer location.

You can see the buffer structure they linked to [here](https://github.com/android/platform_dalvik/blob/android-2.3.7_r1/vm/LinearAlloc.h#L33):

```cpp
  
/\*  
 \* Linear allocation state. We could tuck this into the start of the  
 \* allocated region, but that would prevent us from sharing the rest of  
 \* that first page.  
 \*/  
typedef struct LinearAllocHdr {  
 int curOffset; /\* offset where next data goes \*/  
 pthread\_mutex\_t lock; /\* controls updates to this struct \*/

char\* mapAddr; /\* start of mmap()ed region \*/  
 int mapLength; /\* length of region \*/  
 int firstOffset; /\* for chasing through \*/

short\* writeRefCount; /\* for ENFORCE\_READ\_ONLY \*/  
} LinearAllocHdr;  

```

The team and I had a good time talking about their thought process and whether what they did was a good idea or not as well as critiquing decisions by the [dalvik](http://en.wikipedia.org/wiki/Dalvik_(software)) designers. Some ideas and discussions that came up were:

1. If the size of the application is so big the operating system can't load it, should the application be that big? Why not modularize work? If they couldn't modulraize their application because of how they are reliant on some framework, they should have put proxy's to decouple their logic. Tightly coupled code suffers from this probelm
2. Maybe the internal buffer size was chosen at 5MB to maximize storage space vs estimated application size. It could have been an arbitrary choice to choose that number, OR, it could have been an extremely specific value based on other internal factors not known to us. Either way, messing with that buffer seems like a bad idea.
3. Why not deprecate the application for phones running that operating system? (One of our new engineers brought up that froyo isn't that old, and facebook's capital is based on running their application everywhere, unlike Apple who can dictate when to buy new hardware to run whatever software)
4. Why not have made the internal LinearAlloc buffer dynamic? This got us talking about the memory overhead of having a dynamically resized array (vector) and maybe they chose to not make it dynamic due to performance overhead on an embedded device

We all related to them since I'm sure every engineer has gotten the main business logic to work and then suddenly a show stopping, undocumented, strange heisenbug shows up and you have to stop everything and fix that for weeks.

After that, the team's discussion diverged a little and we got to talking about basic droid development. One of the QA engineers here had dome some droid work before and walked us through a basic application design, describing what is an intent, activity, and how droid views are created.

All in all, a fun tech talk.

