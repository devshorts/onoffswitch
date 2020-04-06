---
layout: post
title: 'Tech talk: B-Trees'
date: 2013-06-07 23:20:51.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- b-tree
- databases
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1555847166;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1043;}i:1;a:1:{s:2:"id";i:3161;}i:2;a:1:{s:2:"id";i:4945;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/06/07/tech-talk-b-trees/"
---
Yesterdays tech talk was on b-trees. B-trees are an interesting tree data structure that are used to minimize disk read access. Also, since they are self balancing, and optimized for sequential reads and inserts, they're really good for file systems and databases. CouchDB, MongoDB, SQLite, SQL Server and other datbases all use either a b-tree or a b+ tree as their data indexes, so it was interesting to discuss b-tree properties.

## Disk io optimizations

A big part of the need for b-trees is disk io optimizations. The need to optimize for disk reads comes from the fact that disk io is slow. Imagine you have to make thousands of reads off disk to try and find a certain piece of data. The rotational delay in a platter drive to read a certain disk block can be up to 20 milliseconds. If you have to read hundreds, or even thousands of blocks of data to find something then your search speed can now range in the hundreds of milliseconds, which is certainly noticeable to a user.

A good way of alleviating some of this disk read time is to have an optimized index that tells you where to go look for your data. So instead of reading blocks off disk to search for data, you can read a much smaller index which points to which disk block the data you want is. Then all you have to do is go find the disk block and do one final search inside of that block to find your data.

## Insertion and deletion

Deletion in a b-tree is cheap and easy. Once you find the data in the index that should be deleted you mark that index location as deleted. Periodically you can maybe purge the data. This is why in many databases when you do a delete the database size doesn't actually get any smaller. With SQLite, for example, you have to issue a `vacuum` command which rebuilds the index. Only at this point does the database get smaller.

Inserts, however, are more complicated. Since the tree needs to be optimized for sequential linear reads, you don't want to use a dense storage for your data. This would mean every time you needed to add something you would have to shift all your data around. Instead, its cheaper and faster to be sparse: leave empty space for yourself to add things to the index.

## Rebalancing

What happens when a block is full? At this point b-trees recursively split themselves. Theoretically, they take the median of the values and create a new split node from that. Half the data goes into the left branch from that node, and half from the right. Look at this trace through from [wikipedia](http://en.wikipedia.org/wiki/B-tree)

![](http://onoffswitch.net/wp-content/uploads/2013/06/B_tree_insertion_example.png)

Here you can see the sparseness of the arrays as well as the final split (the last trace) when 6 (the median) is chosen as the split point and 5 and 7 are moved to separate sparse leaves off the 6 branch.

## B+ Trees

B+ trees, in comparison to b-trees, keep key/value pair data only in the leaf nodes. B-Tree's keep key value pair data at each point in the index. You can see that visualized here

[caption width="569" align="aligncenter"] ![](http://onoffswitch.net/wp-content/uploads/2013/06/image002.jpg) source: http://www.mec.ac.in/resources/notes/notes/ds/bplus.htm[/caption]

## When would you make a B-Tree?

After the discussion of the theory behind b-trees a great question came up: when would you ever use one directly? The conclusion we reached was that you probably wouldn't. If you had so much data you needed to leverage the b tree's disk access properties, you'd probably dump your data into a database that already implemented the tree. You wouldn't really want to build a b-tree from scratch, since there are a lot of optimizations that can be done. For example, oracle apparently [doesn't do a 50-50 data split](http://dba.stackexchange.com/questions/9963/b-tree-node-split-strategy-in-sql-server-for-monotonically-increasing-value) when re-balancing the tree. It instead does a 90-10 split when it detects sequential inserts.

## More reading

Animated trace of the b-tree algorithm: [http://ats.oka.nu/b-tree/b-tree.html](http://ats.oka.nu/b-tree/b-tree.html)

CouchDB explanation of their b-tree implementation: [http://guide.couchdb.org/editions/1/fr/btree.html](http://guide.couchdb.org/editions/1/fr/btree.html)

