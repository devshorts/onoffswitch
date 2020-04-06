---
layout: post
title: 'Tech talk: Bloom Filters'
date: 2013-02-28 16:24:44.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- Bloom filter
- containers
- hash
meta:
  _wpas_done_all: '1'
  _edit_last: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554378326;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4027;}i:1;a:1:{s:2:"id";i:3500;}i:2;a:1:{s:2:"id";i:2274;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/02/28/tech-talk-bloom-filters/"
---
Each Thursday at work my team and I do a 45 minute to an hour discussion on any technical subject that we find interesting. We call these Thursday get togethers tech talks and I think they are awesome. We've been doing them for years and I'm hoping to start reposting our subjects and a blurb about our discussions each week after they happen.

This week's tech talk was about bloom filters. A bloom filter is a memory efficient bit vector that contains multiple hashes of your data. It is a probabilistic data structure. The idea is that it can definitively tell you if a piece of data does NOT exist, but it can't always tell you with certainty that data DOES exist. For more info check out this interactive bloom filter [tutorial](http://billmill.org/bloomfilter-tutorial/).

The reason bloom filters are useful is because it is faster and cheaper to transmit a compressed bit vector that contains maybe/no information than to send an entire hash (or other structured container) of your data set. Even if you aren't sending it anywhere, it's not memory efficient to store all your data for easy querying in a hash or list or whatever. Leave the data in a DB and you can construct a bloom filter representing if data is maybe in the set or definitely not in the set.

Let's imagine a use case. Pretend you want to make a website that queries Wikipedia articles. If you construct a bloom filter of all Wikipedia article titles, then the client (as they search) can test the bloom filter to see if an article maybe exists, or definitely does not exist. If it doesn't exist you can just say "article doesn't exist!" with certainty. If it maybe exists, then you do a query to Wikipedia and either pull back the article or return an empty result set. The advantage here is that you have cut down on a lot of extra processing and network overhead for empty results. Bloom filters usually have a 1% false positive rate. That means that 1% of the time you did work you really didn't need to but that also means that 99% of the time you're actually right!

You may have already encountered bloom filters without even realizing it. Think about registering for a big name site that does immediate validation of available usernames. I'd imagine that instead of doing a SQL query, the ajax call first hits a bloom filter on the backend testing if the username already exists. It's cheaper to give a false positive here to say _username xyz is already taken_ than to pull the data from the database to validate it. Let the user keep picking until you get a definite NO and then you can submit with that.

There are a lot of use cases for bloom filters and they have a lot of interesting variations on the internal mechanisms. By tuning the bit vector size, the hash function choices, and the data you are hashing, you can have a pretty robust maybe/no container set. Cool!

For more info check out these links:

[http://mikecvet.wordpress.com/2010/04/21/bloom-filters/](http://mikecvet.wordpress.com/2010/04/21/bloom-filters/)  
[http://stackoverflow.com/questions/6118154/when-is-a-bloom-filter-useful?rq=1](http://stackoverflow.com/questions/6118154/when-is-a-bloom-filter-useful?rq=1)  
[http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/](http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/)  
[http://zmievski.org/2009/04/bloom-filters-quickie](http://zmievski.org/2009/04/bloom-filters-quickie)  
[http://www.perl.com/pub/2004/04/08/bloom\_filters.html](http://www.perl.com/pub/2004/04/08/bloom_filters.html)  
[http://stackoverflow.com/questions/4282375/what-is-the-advantage-to-using-bloom-filters](http://stackoverflow.com/questions/4282375/what-is-the-advantage-to-using-bloom-filters)

