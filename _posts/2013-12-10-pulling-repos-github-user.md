---
layout: post
title: Pulling back all repos of a github user
date: 2013-12-10 21:20:13.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- git
- ruby
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561055272;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4316;}}}}

permalink: "/2013/12/10/pulling-repos-github-user/"
---
I recently had to relinquish my trusty dev machine (my work laptop) since I got a new job, and as such am relegated to using my old mac laptop at home for development until I either find a new personal dev machine or get a new work laptop. For those who don't know, I'm leaving the DC area and moving to Seattle to work for Amazon, so that's pretty cool! Downside is that it's Java and Java kind of sucks, but I can still do f#, haskell, and all the other fun stuff on the side.

Anyways, since I'm setting up my home dev environment I wanted to pull back all my github repos in one go. If I only had a few of them I would've just cloned them by hand, but I have almost 30 repos, which puts me in the realm of wanting to automate it.

As any good engineer does, I did a quick google and found that someone had written a [ruby script](http://addyosmani.com/blog/backing-up-a-github-account/) to clone all of a users repos using the github API. However, the script is outdated and the github API has changed. It no longer uses YAML, but now JSON, and the URL's are all different.

So, here is the updated script:

[ruby]  
#!/usr/bin/env ruby

require "json"  
require "open-uri"

username = "devshorts"

url = "https://api.github.com/users/#{username}/repos"

JSON.parse(open(url).read).map{|repo|  
 repo\_url = repo["ssh\_url"]

puts "discovered repository: #{repo\_url} ... backing up ..."

system "git clone #{repo\_url}"  
}  
[/ruby]

Unlike the original script, this will clone to the current directory you are in using the same name of each repo (so no renaming it during the clone).

There are a ton of other backup options, but this was fun and simple (and a good way to get me back into using vim)

