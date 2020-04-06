---
layout: post
title: 'Tech Talk: Sorting of ratings'
date: 2013-05-30 15:35:37.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags: []
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561943268;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4191;}i:1;a:1:{s:2:"id";i:3500;}i:2;a:1:{s:2:"id";i:7777;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/30/tech-talk-sorting-ratings/"
---
Today's tech talk discussed different ways to sort ratings system. The topic revolved around a [blog post](http://www.evanmiller.org/how-not-to-sort-by-average-rating.html) we discovered a while ago breaking down different problems with star based sorts.

The article describes a few problems:

## Rating type: Good - Bad

The issue here is that when you use only the difference in positive vs negative ratings, you get skewed results to highly popular (but also maybe highly disliked) items. For example, an item that has upvotes of 200, but downvotes of 50 would have a score of 150. However, an item who has 125 upvotes and no downvotes would be technically scored lower here. The team and I agreed that this isn't a good way of sorting a rating, since the abscense of negatives is a stronger indication of a positive review. I think most people actually do this kind of analysis in their mind: if something is highly rated in both positive and negative, then even though it may be popular, it also could be problematic.

## Rating type: Good / (Good + Bad)

In this scenario, you're taking the average positive reviews of an item. The article breaks this down pretty clearly when it shows how amazon rates an item only 1 review as higher rated than an item with 500 reviews. Like with the previous example, people intuitively know to discount an item with only a few votes: its not statistically significant enough.

## Rating type: Lower bound of Wilson score confidence interval for a Bernoulli parameter

I won't begin to explain what this is, since I don't fully understand all the math behind it, but from what we were able to discern this type of rating system uses a Bernoulli distribution and even outs the weights for both large samples and small samples. It would've been nice to see a comparison of how this would work with the amazon or urban dictionary example, since without that its hard to visualize how this affects the sort. Also I'm not sure how well (or even at all) this works on a rating scale though (1-5 for example), since the article makes the assumption of a binomial distribution (either yes or no).

## Rating type: Baysenian Estimation

In our research for this discussion we also stumbled on [how IMDB does their ratings system](http://wiki.answers.com/Q/What_does_true_Bayesian_estimate_mean_in_connection_with_the_IMDb_Top_250_ratings). In this kind of estimation IMDB weights the actual rating score by the number of ratings. So instead of normalizing, or removing outliers, they adjust the score to lean more towards the average movie rating. This way it takes an exceptional amount of up votes to get something to be rated 10 out of 10 since the score is trying to be weighted towards the average (by both the regular average of the movie AND by the number of votes).

## Conclusion

Our conclusion was that the best way to give meaningful data to a user is to allow for many different kinds of ranking systems. It seems obvious to me that there isn't just one kind of sort that gives a user meaningful data. Different people interpret data differently and are looking for different meanings in the data, so a truly robust system would be able to provide a couple different ways of rating sorts. In the end, here's a relevant xkcd:

[caption id="attachment\_3933" align="aligncenter" width="201"][![From xkcd http://xkcd.com/937/](http://onoffswitch.net/wp-content/uploads/2013/05/tornadoguard.png)](http://xkcd.com/937/) From xkcd http://xkcd.com/937/[/caption]

