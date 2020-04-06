---
layout: post
title: Machine Learning with disaster video posted
date: 2013-09-25 22:06:39.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- F#
- machine learing
- meetup
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560307503;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4209;}i:1;a:1:{s:2:"id";i:4170;}i:2;a:1:{s:2:"id";i:3847;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/09/25/machine-learning-disaster-video-posted/"
---
A few weeks ago we had our second [DC F# meetup](http://www.meetup.com/DC-fsharp/) with speaker [Phil Trelford](http://trelford.com/blog/) where he led a hands on session introducing decision trees. The goal of meetup was to see how good of a predictor we could make of who would live and die on the titanic. [Kaggle](http://www.kaggle.com/c/titanic-gettingStarted) has an excellent data set that shows age, sex, ticket price, cabin number, class, and a bunch of other useful features describing Titanic passengers.

Phil followed [Mathias](www.clear-lines.com/blog/)' format and had an excellent .fsx script that walked everyone through it. I think the best predictor that someone made was close to 84%, though it was surprisingly difficult to exceed that in the short period of time that we had to work on it. I'd implemented my own shannon entropy based ID3 decision tree in C# so this wasn't my first foray into decision tree's, but the compactness of the tree in F# was great to see. On top of that Phil extended the tree to test not just features, but also combinations of features by having the tree accept functions describing features. This was cool and something I hadn't thought of. By the end you sort of had built out a small DSL describing the feature relationships of what you were trying to test. I like it when a problem domain devolves into a series of small DSL like functions!

If anyone is interested Phil let us post all of his slides and information on our [github](https://github.com/DCFsharp/Machine-Learning-From-Disaster). Anyways, here is the video of the session!

<iframe width="420" height="315" src="//www.youtube.com/embed/kh9WjKAG4Jk" frameborder="0" allowfullscreen></iframe>

