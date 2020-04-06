---
layout: post
title: Serialization of lombok value types with jackson
date: 2016-01-26 23:09:33.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- jackson
- lombok
- value
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561483126;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4456;}i:1;a:1:{s:2:"id";i:4919;}i:2;a:1:{s:2:"id";i:4862;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/01/26/serialization-lombok-types-jackson/"
---
For anyone who uses lombok with jackson, you should checkout [jackson-lombok](https://github.com/paradoxical-io/jackson-lombok) which is a fork from [xebia](https://github.com/xebia/jackson-lombok) that allows lombok value types (and lombok generated constructors) to be json creators.

The original authors compiled their version against jackson-core 2.4.\* but the new version uses 2.6.\*. Props needs to go to github [user kazuki-ma](https://github.com/kazuki-ma) for submitting a PR that actually addresses this. Paradoxical just took those fixes and published.

Anyways, now you get the niceties of being able to do:

[java]  
@Value  
public class ValueType{  
 @JsonProperty  
 private String name;

@JsonProperty  
 private String description;  
}  
[/java]

And instantiate your mapper:

[java]  
new ObjectMapper().setAnnotationIntrospector(new JacksonLombokAnnotationIntrospector());  
[/java]

Enjoy!

