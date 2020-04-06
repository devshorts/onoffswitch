---
layout: post
title: Building better regular expressions
date: 2013-05-06 08:00:12.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- regex
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561472667;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4343;}i:1;a:1:{s:2:"id";i:4316;}i:2;a:1:{s:2:"id";i:4068;}}}}

permalink: "/2013/05/06/composable-regex/"
---
Every software developer has at one point in time heard the adage

> If you have a problem and you think you can solve it with [threads|pointers|regex|etc], now you have two problems

For me, I've always told it with regex (and I think that's the [official](http://www.codinghorror.com/blog/2008/06/regular-expressions-now-you-have-two-problems.html) way to do it). It's not that threads and pointers aren't hard, but more that with proper stylistic choices and with experience, they can be easily manageable and simple to debug. Regex though, have a tendency to spiral out of control. What starts with something simple always bloats into an enormously difficult to read haze of PERLgasms.

For example, I frequently wonder why in the 21st century why we still deal with a syntax like this:

[code]  
(?:(?:\r\n)?[\t])\*(?:(?:(?:[^()\<\>@,;:\\".\[\] \000-\031]+(?:(?:(?:\r\n)?[\t]  
)+|\Z|(?=[\["()\<\>@,;:\\".\[\]]))|"(?:[^\"\r\\]|\\.|(?:(?:\r\n)?[\t]))\*"(?:(?:  
\r\n)?[\t])\*)(?:\.(?:(?:\r\n)?[\t])\*(?:[^()\<\>@,;:\\".\[\] \000-\031]+(?:(?:(  
?:\r\n)?[\t])+|\Z|(?=[\["()\<\>@,;:\\".\[\]]))|"(?:[^\"\r\\]|\\.|(?:(?:\r\n)?[  
\t]))\*"(?:(?:\r\n)?[\t])\*))\*@(?:(?:\r\n)?[\t])\*(?:[^()\<\>@,;:\\".\[\] \000-\0  
31]+(?:(?:(?:\r\n)?[\t])+|\Z|(?=[\["()\<\>@,;:\\".\[\]]))|\[([^\[\]\r\\]|\\.)\*\  
](?:(?:\r\n)?[\t])\*)(?:\.(?:(?:\r\n)?[  
[/code]

Even the most seasoned engineers couldn't tell me what this did off the bat.

Still, regex, even for all their annoyances, are an extremely powerful form of [deterministic finite automata](http://en.wikipedia.org/wiki/Deterministic_finite_automaton). I think it's important to know what they are and how to use them, and when you wield them properly they can make life better.

## Let's make it better

I'm also not the only person to have given it some thought. [Martin Fowler](http://martinfowler.com/bliki/ComposedRegex.html) has also discussed the regex composability problem. Fowler says

> One of the most powerful tools in writing maintainable code is break large methods into well-named smaller methods - a technique Kent Beck refers to as the Composed Method pattern.

And I absolutely agree. The problem is that writing regex usually starts out this way too. You pattern match on one subset, then you figure out another pattern match group, and you start to build yourself a long and complex regex string. However, by the end, even you can't figure out what it does. The human eye can't distinguish the logical patterns anymore.

While others have also tackled the issue, their solutions usually involved special classes and workflows. I initially thought about building a full fledged DSL, mimicking those solutions, but came up with a simpler approach. I don't want to replace regex completely, I just want a way to view my composable groups easier.

My regex patterns use white space ignored key value pairs for composable groups, and the very last line of the regex (where all the key value pairs are seperated by newlines) will be the final composed regex. In the end, the regular expressions now look something like this:

[code]  
group1 = [A-f]  
group2 = [g-z]

group1|group2  
[/code]

And build out into

[code]  
[A-f]|[g-z]  
[/code]

## The builder

The code to do this is pretty trivial. It's all simple string manipulation. Let me show you anyways:

[csharp]  
public class Regexer  
{  
 public String Regex { get; private set; }

public List\<String\> DebugTrace { get; private set; }

public Regexer(string regex, bool includesComments = true, bool useDebug = true)  
 {  
 if (String.IsNullOrEmpty(regex))  
 {  
 throw new ArgumentNullException("regex", "Regex cannot be null");  
 }

DebugTrace = new List\<string\>();

Parse(regex, includesComments, useDebug);  
 }

private void Parse(string regex, bool includeComments, bool useDebug)  
 {  
 var splits = regex.Split(new[] { "\r\n", "\n" }, StringSplitOptions.RemoveEmptyEntries);  
 if (splits.Count() == 1)  
 {  
 Regex = splits.First();  
 return;  
 }

var groups = (from item in splits.Take(splits.Count() - 1)  
 where !item.StartsWith("##")  
 let keyValueSplit = item.Split(new[] { '=' }, 2, StringSplitOptions.RemoveEmptyEntries)  
 let key = keyValueSplit.First().Trim()  
 let value = keyValueSplit.Last().Trim()  
 where !String.IsNullOrEmpty(key)  
 where !String.IsNullOrEmpty(value)  
 let regexWithoutComments = includeComments ? value.Split(new[] { "##" }, 2, StringSplitOptions.RemoveEmptyEntries).First().Trim() : value  
 select new { Key = key, Value = regexWithoutComments }).ToList();

var final = splits.Last().Trim();

Regex = final;

for (int i = groups.Count - 1; i \>= 0; i--)  
 {  
 var item = groups[i];  
 Regex = Regex.Replace(item.Key, item.Value);  
 if (useDebug)  
 {  
 DebugTrace.Add(Regex);  
 }  
 }  
 }  
}  
[/csharp]

Basically build out all key/value pairs and string replace them going from the bottom of the call group up. The string replace also supports comments, by adding in "##" followed by whatever you want to comment.

## Validation of email addresses

To test this idea, I decided it would be fun to try to validate email addresses. For those who don't' know, the [email RFC](http://tools.ietf.org/html/rfc3696#page-5) is absurdly complex. In fact the initial regex I posted is just a very small snippet of the official [email validation regex](http://ex-parrot.com/~pdw/Mail-RFC822-Address.html).

I didn't think to account for ALL of those cases, but I was able to get a pretty robust email verifier on almost the first try. Writing regex in human readable groups really does make a difference.

Here is the composed regex I built:

[code]  
weirdChars = (!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)  
numbers = \d  
characters = [A-z]  
anyChars = (weirdChars|numbers|characters)

lettersFollowedBySingleDot = (anyChars+\.anyChars+)

names = anyChars|lettersFollowedBySingleDot

onlyQuotableCharacters = @|\s  
quotedNames = ""(names|onlyQuotableCharacters)+""

anyValidStart = (names|quotedNames)+

group = (quotedNames:anyValidStart)|anyValidStart

local = ^(group)

ipv4 = ((\d{1,3}.){3}(\d{1,3}))

ipv6Entry = ([a-f]|[A-F]|[0-9]){4}? ## group of 4 hex values  
ipv6 = ((ipv6Entry:){7}?ipv6Entry) ## 8 groups of ipv6 entries

comAddresses = (characters+(\.characters+)\*) ## stuff like a.b.c.d etc  
domain = (comAddresses|ipv6|ipv4)$ ## this has to be at the end

(local)@(domain)  
[/code]

Which compiles to:

[csharp]  
(^(("(((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])|(((!|-|\+|\\|\$|\^|~|#|%  
|\?|{|}|\_|/|=)|\d|[A-z])+\.((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])+)|@|  
\s)+":(((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])|(((!|-|\+|\\|\$|\^|~|#|%  
|\?|{|}|\_|/|=)|\d|[A-z])+\.((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])+)|"((  
(!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])|(((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_  
|/|=)|\d|[A-z])+\.((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])+)|@|\s)+")+)|(  
((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])|(((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|  
\_|/|=)|\d|[A-z])+\.((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])+)|"(((!|-|\+|  
\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])|(((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d  
|[A-z])+\.((!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)|\d|[A-z])+)|@|\s)+")+))@((([A-z]+  
(\.[A-z]+)\*)|((([a-f]|[A-F]|[0-9]){4}?:){7}?([a-f]|[A-F]|[0-9]){4}?)|((\d{1,3}.)  
{3}(\d{1,3})))$)  
[/csharp]

And passes all of these tests:

[csharp]  
[Test]  
[TestCase("someDude@gmail.com", true)]  
[TestCase("foo", false)]  
[TestCase("foo@", false)]  
[TestCase("!@2001:0db8:85a3:0000:0000:8a2e:0370:7334", true)]  
[TestCase("!@2001:0db8:85a3:0000:0000:8a2e:0370:73345", false)]  
[TestCase("stupidM0nk3yA0l@aol.net", true)]  
[TestCase("stupidM0nk3yA0l@aol.net.net", true)]  
[TestCase("123+guy@domain", true)]  
[TestCase("@@domain", false)]  
[TestCase("!@ 2001:0db8:85a3:0000:0000:8a2e:0370:73345", false)]  
[TestCase(@"""Abc\@def""@example.com", true)]  
[TestCase("customer/department=shipping@example.com", true)]  
[TestCase("$A12345@example.com", true)]  
[TestCase("!def!xyz%abc@example.com", true)]  
[TestCase("\_somename@example.com", true)]  
[TestCase("abc+\"foo=123$\"+test@example.com", true)]  
[TestCase("", false)]  
[TestCase("john@uk", true)]  
[TestCase("john.a@uk", true)]  
[TestCase("john.@uk", false)]  
[TestCase("john..@uk", false)]  
[TestCase(".john@uk", false)]  
[TestCase("\"a group\":cal@iamcalx.com", true)]  
[TestCase("\"a group\":@iamcalx.com", false)]  
[TestCase(":\"a group\":cal@iamcalx.com", false)]  
[TestCase("\"a group\":cal@192.168.1.1.1", false)]  
[TestCase("\"a group\":cal@192.168.1.1", true)]  
[TestCase("\"a group\":cal@192.168.1.100", true)]  
[TestCase("\"a group\":cal@192.168.1111.100", false)]  
public void Email(string email, bool pass)  
{  
 var composableRegex = @"

weirdChars = (!|-|\+|\\|\$|\^|~|#|%|\?|{|}|\_|/|=)  
 numbers = \d  
 characters = [A-z]  
 anyChars = (weirdChars|numbers|characters)

lettersFollowedBySingleDot = (anyChars+\.anyChars+)

names = anyChars|lettersFollowedBySingleDot

onlyQuotableCharacters = @|\s  
 quotedNames = ""(names|onlyQuotableCharacters)+""

anyValidStart = (names|quotedNames)+

group = (quotedNames:anyValidStart)|anyValidStart

local = ^(group)

ipv4 = ((\d{1,3}.){3}(\d{1,3}))

ipv6Entry = ([a-f]|[A-F]|[0-9]){4}? ## group of 4 hex values  
 ipv6 = ((ipv6Entry:){7}?ipv6Entry) ## 8 groups of ipv6 entries

comAddresses = (characters+(\.characters+)\*) ## stuff like a.b.c.d etc  
 domain = (comAddresses|ipv6|ipv4)$ ## this has to be at the end

(local)@(domain)";

var regex = new Regexer(composableRegex).Regex;

Console.Write(regex);

if (pass)  
 {  
 Assert.IsTrue(Regex.IsMatch(email, regex));  
 }  
 else  
 {  
 Assert.IsFalse(Regex.IsMatch(email, regex));  
 }  
}  
[/csharp]

At one point I had a problem with my ipv4 vs ipv6 validation, but now it was trivial to test. For the domain I could remove both `comAddresses` and `ipv6` and deal only with ipv4. Without having this kind of composable group I'd have to manually edit the enormous regex and it'd be easy to make a parenthesis mistake.

## Try it out!

At the suggestion of a coworker, I've uploaded a very small MVC4 test app to generate regex for you using this. Go to [composableregex.apphb.com](http://composableregex.apphb.com/) to try it out. Big thanks to Carlo for helping make it look nice!

I went to [debuggex.com](http://www.debuggex.com/) which is a new regular expression visualizer and ran my regex in it. Here's the breakdown. FYI, the image is pretty big

[![regex_visualizer](http://onoffswitch.net/wp-content/uploads/2013/05/regex_visualizer-216x600.png)](http://onoffswitch.net/wp-content/uploads/2013/05/regex_visualizer.png)

## Conclusion

It's too bad the regex spec doesn't let us build things out like this by default, though it is reasonably easy to build your own regex composer. While I don't think my solution is maybe the most elegant, or high tech, it does help me minimize mistakes by being able to work on smaller chunks at a time.

