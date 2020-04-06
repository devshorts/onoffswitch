---
layout: post
title: Working on a long term svn branch
date: 2013-06-05 17:13:50.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rants
tags:
- branches
- svn
- version control
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559886835;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4892;}i:1;a:1:{s:2:"id";i:3847;}i:2;a:1:{s:2:"id";i:3367;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/06/05/working-long-term-svn-branch/"
---
I work on a reasonably small team and for the most part everyone works in trunk. But it can happen where you need to switch over to a long term feature branch (more than a week or two) that can last sometimes months. The problem here is that your branch can easily diverge from trunk. If the intent is that the feature branch will eventually become the master (trunk) then you should merge the feature branch frequently. For me, this method has worked really well.

Merging often lets you take the trunk fixes that happen and you manually resolve any conflicts as they come in. Since the feature branch is going to be the final thing (when the feature is done), svn needs to know how to deal with these conflicts. It's _much_ better to deal with them as they come in, rather than try to integrate a feature branch after months of work only to see an svn merge with hundreds of conflicts.

The problem with resolving those conflicts later is that contextually you can't remember what they were doing anymore. If you have a conflict that spans 2 or 3 files, it's easy to get lost in what needs to be discarded, what needs to be modified, and what needs to be resolved with local or repo changes. This just means that your QA is going to absolutely hate you because nobody is confident that the merge was complete: something could be missing, or a logical piece isn't right. By merging frequently from trunk into the branch you make svn's job easier. It knows how to resolve potential conflicts because you already did it.

You can take this one step further and do the same thing with multiple feature branches. Lets say you have a setup like this:

![svn](http://onoffswitch.net/wp-content/uploads/2013/06/svn1.jpg)

You have two feature branches and trunk. Periodically you should merge the first branch from trunk (I do this every monday morning). Then periodically also merge the second branch _from the first branch_. When the first branch is done, you can easily reintegrate it.

After you integrate the first branch, you can start to merge the second branch back off of trunk

![svn2](http://onoffswitch.net/wp-content/uploads/2013/06/svn21.jpg)

Eventually when the second branch is done, you can reintegrate it back into trunk and you won't have any conflicts.

The important thing here is to do your due diligence in making sure all the conflicts and merges are properly done and done often. Don't wait till the last minute to do this, it can be time consuming but it's a lot easier to do this upfront then all at the end.

