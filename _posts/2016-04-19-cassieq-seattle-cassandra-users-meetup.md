---
layout: post
title: CassieQ at the Seattle Cassandra Users Meetup
date: 2016-04-19 16:21:30.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- cassandra
- cassieq
- meetup
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559832373;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4783;}i:1;a:1:{s:2:"id";i:4750;}i:2;a:1:{s:2:"id";i:4800;}}}}

permalink: "/2016/04/19/cassieq-seattle-cassandra-users-meetup/"
---
Last night Jake and I presented [CassieQ](https://github.com/paradoxical-io/cassieq) (the distributed message queue on cassandra) at the seattle cassandra users meetup at the Expedia building in Bellevue. Thanks for everyone who came out and chatted with us, we certainly learned a lot and had some great conversations regarding potential optimizations to include in CassieQ.

A couple good points that came up where how to minimize the use of compare and set with the monoton provider, whether we can move to time UUID's for "auto" incrementing monotons. Another interesting tidbit was the discussion of using potential time based compaction strategies that are being discussed that could give a big boost given the workflow cassieq has.

But my favorite was the suggestion that we create "kafka" mode and move the logic of storing pointer offsets out of cassieq and onto the client, in which case we could get enormous gains since we no longer need to do compare and sets for multiple consumers. If we do see that pull request come in I think both Jake and I would be pretty stoked.

Anyways, the slides of our presentation are available here: [paradoxical.io/slides/cassieq](paradoxical.io/slides/cassieq) ([keynote](https://github.com/paradoxical-io/paradoxical.io/blob/gh-pages/slides/cassieq/cq.key))

