---
layout: post
title: 'Tech Talk: Path finding algorithms'
date: 2013-05-09 17:35:38.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- bfs
- graphs
- path finding
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558686525;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4027;}i:1;a:1:{s:2:"id";i:4783;}i:2;a:1:{s:2:"id";i:3656;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/09/tech-talk-path-finding-algorithms/"
---
Today's tech talk was about path finding algorithms. The topic was picked because of a recent linked shared to reddit that [visualized](http://qiao.github.io/PathFinding.js/visual/) different algorithms. The neat thing about the link is that you can really see how different algorithms and heuristics modify the route.

In general, path finding algorithms are based off a breadth first search. At each iteration while walking through the graph you check the nearest neighbors you update what was the calculated weight of the path to get to that neighbor. If it was cheaper to get to the neighbor via your own node (than whoever visited it previously) you update the neighbors weight to reflect that. This is pretty much [dijsktras algorithm.](http://en.wikipedia.org/wiki/Dijkstra's_algorithm) Disjkstra gives you the shortest path cost, but not necessarily the shortest path. To find the shortest path you mark each node with who its cheapest parent is (i.e. the node you need to get to the neighbor). When you get to the final destination all you have to do is backtrace from the parent reference until you get back to the source.

You can also use a priority queue (implemented as a min heap) to store who is the best neighbor to check next, since each time you check a neighbor you add them to a list of checked (but unvisited) nodes called the "open list". By ordering the open list with the cheapest neighbor you can more effectively process who to check next.

A couple of neat things we discussed were the differences in algorithms. Almost all of them were based off of Dijskstra, except instead of just using the weight of the edge to calculate the distance to a node, the other algorithms also used some sort of heuristic to guide the direction of neighbor checking. For example, with A\* you add the distance from the neighbor to the target to the total weight of a node. With best-first, you add the distance from the neighbor to the target, but you also amplify the distance weight by a large factor (making it like a focused A\*).

There are different kinds of [distance calculations](http://lyfat.wordpress.com/2012/05/22/euclidean-vs-chebyshev-vs-manhattan-distance/) that you use as the heuristic weight too. Manhattan distance counts only vertical and horizontal movements (like a taxi in manhattan). So a diagonal move would cost you two units, since you have to move once horizontally, and once vertically. Euclidean distance is a vector difference between to the two x,y coordinates. And Chebyschev distance is basically the maximum of either direction (x, or y). On a graph where all units are one, a euclidean distance of going diagonally is sqrt 2, Chebyschev is 1, and Manhattan is 2.

We also talked about [jump point search](http://zerowidth.com/2013/05/05/jump-point-search-explained.html), which is completely different. The idea is to use a set of rules to eliminate nodes you don't need to check. This makes it much more effective by being able to remove huge swaths of the search space. The downside here is that it only works if you can check cells beyond the reach of where you are at. If you need to actually traverse a cell to find its weight you can't really use jump point.

After that we got into a discussion of how 3 dimensional path finding algorithms work. With real world robotics you can't do a direct BFS of every single point in space around you, it would take forever. So, instead what happens is you probabilistically select N number of points in the real world space. This gives you a random sampling of what is out there in the world view, and you create a graph using that. At that point you can use BFS to find the nearest path and take a step in that direction. At each step, you build a new graph and try again. This gives you a discrete set of sample points to work in and generally moves you in the right direction.

A coworker also mentioned a paper discussing when the target moves. When the target moves you can treat your graph as changing vector field. Again, at each time interval you re-evaluate what is the best path to the target and make a move in that direction. See [figure 4 (PDF)](http://students.cs.byu.edu/~cs470ta/goodrich/fall2004/lectures/Pfields.pdf).

In the end path finding is an extremely interesting subject with lots of ways of doing it.

