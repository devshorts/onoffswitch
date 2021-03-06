---
layout: post
title: Dont be afraid of dependency updates
date: 2016-11-25 22:03:20.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rants
tags:
- dependencies
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpcom_is_markdown: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561858553;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4764;}i:2;a:1:{s:2:"id";i:4028;}}}}
  _wpas_done_all: '1'

permalink: "/2016/11/25/dont-afraid-dependency-updates/"
---
Lots of place I've worked at have had an irrational fear of upgrading their dependencies. I understand why, when you have something that `works` you don't want to rock the boat. You want to focus on building your product, not dealing with potential runtime errors. Your ops team is happy, things are stable. Life is great.

However, just like running from your problems, freezing your dependencies is a recipe for disaster. Just like normal software maintenance, your dependencies MUST be upgraded on a regular basis. It sucks, nobody likes dealing with weird transitive issues, but without a regular upgrade schedule (every 6 months to a year at minimum) you run the risk of realizing that you can't upgrade at all!

This is a crappy place to be in, and you know when you're there because you try and pull in some updated library that has the features you want and/or need and everything either fails to compile, blows up at runtime, and you end up with a giant mishash of dependency exclusions and staring fruitless at dependency graphs trying to figure out "if I pick one minor version down of this and one minor version up of that maaaaybe it'll work"

What managers and sometimes even leads don't understand is that without staying on top of this, your cadence will slow down. The first few years you won't notice, but if you let it stagnate, come 3, 4, or 5 years later it will be very hard to update. Without updates you're missing on security fixes, industry standard changes, performance boosts, bug fixes, etc.

I'm not advocating for staying bleeding edge, but it is worth staying up to date. There is a difference. Bleeding edge is usually alphas and betas of libraries/products, ones that haven't been battle tested or settled down (maybe the API is constantly in flux, looking at you angular 2.0). But stable releases should be moved onto. Your team needs to have a plan for upgrading, and isolating dependencies and changes. You need to be able to silo projects so that upgrades in one place don't require major cascading upgrades somewhere else. If you run into that, unfortunately you have a poorly factored ecosystem that needs to be trimmed and decoupled.

And if you do find yourself in this situation, especially on something you inherited. I feel you. Trust me.

