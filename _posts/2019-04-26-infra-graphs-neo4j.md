---
layout: post
title: Infra graphs with neo4j
date: 2019-04-26 21:51:12.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags: []
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561804019;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4899;}i:1;a:1:{s:2:"id";i:4673;}i:2;a:1:{s:2:"id";i:4167;}}}}
  _wpas_done_all: '1'

permalink: "/2019/04/26/infra-graphs-neo4j/"
---
I spent some time recently mucking around with neo4j attempting to model infrastructure, incidents, teams, users, etc. Basically what does it take to answer questions about organizations.

Getting neo4j set up with go was non trivial and thankfully someone had documented how to do it already (instructions in the readme: [https://github.com/devshorts/graphql](https://github.com/devshorts/graphql)). In the sample API I exposed we can

- Find related incidents. The pathway here is incidentA is failing because of infraA. incidentB is failing because of infraB. InfraB depends on some pathway that ends up infraA. This means that from InfraB -\> InfraA there is a relationship, and so that implies that IncidentA and IncidentB are related. 
- Find betweeness of the graph. This shows graph nodes that have heavy flow (high connections) and can be potential hot spots
- Find communities in the graph. This shows clusterability of infrastructure/teams/etc.

The API exposed in the github is meant to model dynamically creating incidents and adding semantic links. So for example, you can post an incident to the `/incidents` api and then add links to the incident (users/failing infra/etc) via the `/links` api. As you add links you can query for related incidents and then find pathways from your incident to another.

Pretty neat!

Included in the project is a way to build a sample graph:

![](http://onoffswitch.net/wp-content/uploads/2019/04/graph-6.png)

