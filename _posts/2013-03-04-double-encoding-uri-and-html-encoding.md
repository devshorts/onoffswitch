---
layout: post
title: 'Double encoding: URI and HTML encoding'
date: 2013-03-04 16:30:18.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags: []
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _oembed_6073140669e937827d5610641b40cdf4: "{{unknown}}"
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561033504;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4262;}i:1;a:1:{s:2:"id";i:3942;}i:2;a:1:{s:2:"id";i:4725;}}}}
  _oembed_dcc58c858c7066130a1a8a543b535881: "{{unknown}}"
  _oembed_da0fe90d451adcefeea5c7bb0f51ab34: "{{unknown}}"
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/04/double-encoding-uri-and-html-encoding/"
---
URL's have specific characters that are special, like `%` and `&` that if you need to use as part of your GET URI then you need to [encode](http://www.w3schools.com/tags/ref_urlencode.asp) them. For example:

[html]  
http://localhost?key=this & that&key2=value2  
[/html]

It's obvious that this URL is invalid, `this & that` has both spaces and a special character `&`. In fact, you may have even noticed it's invalid in your browser:

[caption id="attachment\_3126" align="alignnone" width="432"] ![Screenshot of what you see above. Yes, I know it's a little meta](http://onoffswitch.net/wp-content/uploads/2013/03/invalidurl2.png) Screenshot of what you see above. Yes, I know it's a little meta[/caption]

To get around this you can URI encode your URL's which will convert special characters to ASCII:

[html]  
http://localhost?key=this%20%26%20that&key2=value2  
[/html]

But there is also [HTML encoding](http://en.wikipedia.org/wiki/Character_encodings_in_HTML), which means escaping special HTML characters when you are putting in text into html. For example:

[html]  
\<p\>6 \> 7\</p\>  
[/html]

Doesn't work, since `>` is a special character. To get around this, you need to HTML encode your text. HTML encoding uses special characters to indicate escaping of text. HTML encoding the above example would make your text

[html]  
\<p\>6 &gt; 7\</p\>  
[/html]

But sometimes you want to put in some HTML dynamically into a page that also contains a URL. Here you need to double encode (encode the URI and the HTML). The ordering here matters. Let's take this HTML text as the source:

[html]  
\<a href="me.aspx?Filename=Anton's Document" /\>  
[/html]

And lets encode it first with HTML then with URI:

[html]  
html encoded: \<a href="me.aspx?Filename=Anton&apos;s+Document" /\>  
uri encoded: \<a href="me.aspx?Filename=Anton%26apos;s+Document" /\>  
[/html]

But what if we do it the other way around?

[html]  
uri encoded: \<a href="me.aspx?Filename=Anton's+Document" /\>  
html encoded: \<a href="me.aspx?Filename=Anton&apos;s+Document" /\>  
[/html]

See the difference? Look at where the apostrophe would be

[code]  
invalid: %26apos;  
valid: &apos;  
[/code]

The first example will give you an invalid URL but the second example is the URL you want. URL and HTML encodings aren't interchangable, they are used for specific scenarios and sometimes need to be used together (in the right order).

