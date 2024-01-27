---
layout: post
title: Merging two immutable dictionaries in F#
date: 2013-03-18 21:20:47.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- F#
- Utilities
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560791954;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3536;}i:1;a:1:{s:2:"id";i:4365;}i:2;a:1:{s:2:"id";i:3847;}}}}

permalink: "/2013/03/18/merging-two-dictionaries-in-f/"
---
If you ever need to merge two immutable dictionaries (maps) that may share the same key, here is how I did it

```fsharp
  
let mapMerge group1 group2 appender =  
 group1 |\> Seq.fold(fun (acc:Map\<'a,'b\>) (KeyValue(key, values)) -\>  
 match acc.TryFind key with  
 | Some items -\> Map.add key (appender values items) acc  
 | None -\> Map.add key values acc) group2

// for example, assume map1 and map2 are dictionaries of type  
// Map\<string, seq\<string\>\>  
// then the appender deals with how to merge duplicate keys  
let joinMaps = mapMerge map1 map2 Seq.append  

```

It doesn't matter which group you treat as the source and which group you treat as the seed since you are creating a new dictionary out of the two. By pre-seeding the fold with one of the dictionaries you know that the accumulator will already have some of the values you want. The map will iterate over the second dictionary (used as the source). All you need to do is pull out the data from the dictionary if it exists, and if so return a new dictionary that adds the elements to the accumulator again. By injecting a custom "key conflict" resolver you can merge any kind of item. For my example, I have a map of type

```fsharp
  
Map\<string, seq\<string\>\>  

```

Which is why I injected the `Seq.append` function. The types don't need to be specified in the sequence fold, I just put them there to be explicit for the post.

`KeyValuePair` is a built in discriminated union that you can use to break up a key value pair tuple that the map iterator gives you.

After figuring this out I realized I could have saved myself a little bit of time if I had just googled it. If I had done that I would've found this [great question](http://stackoverflow.com/questions/3974758/in-f-how-do-you-merge-2-collections-map-instances) on stackoverflow, which shows a few other methods and basically exactly what I have (though not all preserve key conflicts). Hey, great minds think alike, right?

