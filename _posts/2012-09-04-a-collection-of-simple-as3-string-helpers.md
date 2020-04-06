---
layout: post
title: A collection of simple AS3 string helpers
date: 2012-09-04 14:21:36.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- AS3
- Utilities
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '830834780'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559667323;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4862;}i:2;a:1:{s:2:"id";i:2365;}}}}

permalink: "/2012/09/04/a-collection-of-simple-as3-string-helpers/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/a-collection-of-simple-as3-string-helpers/)_

We all know that it's smart to create helper and utility classes when faced with a problem that can be encapsulated, but sometimes we forget that even the smallest of things can be put into a helper. Whenever you find yourself writing something more than once you should think about encapsulating that logic. Even small logical elements should be moved to a separate function. It helps with readability, maintainability, and a separation of logic. It also makes things easier to test. Here are a couple of ActionScript string utilities we use. We have tons of them and I'll be posting snippets here and there of ones we find useful.

## Check if a list is empty.

I use this one everywhere! It seems silly, but this may be the most used helper function in our entire application.

[csharp]  
public static function isEmpty(list:IList):Boolean {  
 return list == null || list.length == 0;  
}  
[/csharp]

## Flatten a list into a delimited string

It's handy to be able to say given a list of objects, print out a comma (or delimiter) seperated string representing that list. This function takes a list, a function that formats each item, and an optional delimiter. An example usage is:

[csharp]  
var foldedString:String = foldToDelimitedList(listOfUsers,  
 function(item:Object):String{  
 return (item as UserData).userName;  
 });  
[/csharp]

Which would give you something like "user1, user2, user3".

[csharp]  
public static function foldToDelimitedList(vals:ArrayCollection, formatter:Function, delim:String = ", "):String{  
 var retString:String = "";  
 var count:int = 0;  
 for each(var item:Object in vals){  
 retString += formatter(item);

count++;

if(count \< vals.length){  
 retString += delim;  
 }  
 }  
 return retString;  
}  
[/csharp]

## Find an item in a list

This one is handy when you want to know if something is in a list based on a certain property. If it finds the item it will return to you the index it found. You use it like this:

[csharp]  
var index:int = findItem(list, "someProperty", "expectedPropertyValue");  
[/csharp]

For an element whose property `someProperty` matches the value `expectedPropertyValue`, it will return the first found index.

[csharp]  
public static function findItem(dataProvider:Object, propName:String, value:Object, useLowerCase:Boolean = false):int {

if (value == null) {  
 return -1;  
 }

var max:int = dataProvider.length;  
 if (useLowerCase) {  
 value = value.toString().toLocaleLowerCase();  
 }  
 for(var i:int=0; i\<max; i++) {  
 var item:Object = dataProvider[i];

if (item == null) {  
 continue;  
 }  
 var loopValue:Object = item[propName];  
 if (loopValue == null) {  
 continue;  
 }

if (loopValue == value || (useLowerCase && loopValue.toString().toLocaleLowerCase() == value)) {  
 return i;  
 }  
 }  
 return -1;  
}  
[/csharp]

