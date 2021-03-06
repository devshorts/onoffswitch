---
layout: post
title: RMQ failures from suspended VMs
date: 2016-02-04 21:18:29.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- failure
- partition
- rmq
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1546394009;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2635;}i:1;a:1:{s:2:"id";i:4945;}i:2;a:1:{s:2:"id";i:4737;}}}}

permalink: "/2016/02/04/rmq-failures-suspended-vms/"
---
My team recently ran into a bizarre RMQ partition failure in a production cluster. RMQ doesn't handle partition failures well, and while you can set up auto recovery (such as suspension of minority groups) you need to manually recover from it. The one time I've encountered this I got a very useful message in the admin managment page indicating that parts of the cluster were in partition failure, but this time things went weird.

Symptoms:

- Could not gracefully restart rmq using `rabbitmqctl stop_app/start_app`. The commands would stall
- Could not list queues for any vhost. `rabbitmqctl list_queues -p [vhost]` would stall
- Logs showed partition failure
- People could not consistently log into the admin api without stalls, or other strange issues even when clearing browsing data/local storage/incognito/different browsers
- Rebooting the master did not help

In the end the solution was to do an NTP time sync, turn off all clustered slaves (shut down their VM's, not go into suspension). Once that occurred, the master was rebooted and then it stabilized. After that we brought up each slave one by one until it went green.

Anyways, figured I'd share the symptoms and the solution in case anyone else runs into it.

