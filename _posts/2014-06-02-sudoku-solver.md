---
layout: post
title: Constraint based sudoku solver
date: 2014-06-02 08:00:29.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- algorithms
- c#
- sudoku
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554788393;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:4914;}i:2;a:1:{s:2:"id";i:3016;}}}}

permalink: "/2014/06/02/sudoku-solver/"
---
A few weekends ago I decided to give solving [Sudoku](http://en.wikipedia.org/wiki/Sudoku) a try. In case you aren't familiar with Sudoku, here is what an unsolved board looks like

[caption width="364" align="aligncenter"] ![](http://onoffswitch.net/wp-content/uploads/2014/06/364px-Sudoku-by-L2G-20050714.svg_.png) from wikipedia[/caption]

And here is a solved one

[caption width="364" align="aligncenter"] ![](http://onoffswitch.net/wp-content/uploads/2014/06/364px-Sudoku-by-L2G-20050714_solution.svg_.png) from wikipedia[/caption]

Sudoku, of size 3 is pretty easy. Make a snapshot of the board, pick a random open cell, find out what its available possibilities are and set it to a value. To figure out it's possibilities you need get the cells "group". This means all the values of the 3x3 cell it's in, as well as all the values of the row that it's in and the columns that it's in.

Based on what is available, you can choose a number that isn't taken, plop it in down, and then recursively repeat. If nothing is available, and the board isn't empty, you messed up and the recursion will backtrack.

Let's get solvin'

## Some helper functions

Let's assume the board is a 2 dimensional nullable integer array, and that we have a class called `Location` that just encapsulates an (x,y) tuple:

[csharp]  
public int? Get(int x, int y)  
 {  
 if (x \> \_board.Length || y \> \_board.Length)  
 {  
 throw new Exception("invalid position");  
 }

return \_board[x, y];  
 }

public void Set(Location location, int value)  
 {  
 \_board[location.X, location.Y] = value;

for (int i = 0; i \< \_emptySpaces.Count; i++)  
 {  
 if (\_emptySpaces[i].X == location.X && \_emptySpaces[i].Y == location.Y)  
 {  
 \_emptySpaces.RemoveAt(i);  
 return;  
 }  
 }  
 }  
[/csharp]

Easy enough. Let's also keep track of empty spaces as we set things since we'll want to be able to query for empty spaces later (rather than finding them), and have a wrapper to update values of the board.

Now lets make sure we can get all the information regarding a cell's group. This will be relevant for our calculations. It's a lot of boring boilerplate, but here it is:

[csharp]  
public IEnumerable\<int\> UsedNumbersInSpace(Location location)  
 {  
 int x = location.X;  
 int y = location.Y;

foreach (var item in GetCol(x, y))  
 {  
 if (item.HasValue)  
 {  
 yield return item.Value;  
 }  
 }

foreach (var item in GetRow(x, y))  
 {  
 if (item.HasValue)  
 {  
 yield return item.Value;  
 }  
 }

foreach (var item in GetSquare(x, y))  
 {  
 if (item.HasValue)  
 {  
 yield return item.Value;  
 }  
 }  
 }

private IEnumerable\<int?\> GetRow(int x, int y)  
 {  
 for (int i = 0; i \< N \* N; i++)  
 {  
 yield return Get(i, y);  
 }  
 }

private IEnumerable\<int?\> GetCol(int x, int y)  
 {  
 for (int i = 0; i \< N \* N; i++)  
 {  
 yield return Get(x, i);  
 }  
 }

private IEnumerable\<int?\> GetSquare(int x, int y)  
 {  
 int xStart = x - (x % N);  
 int yStart = y - (y % N);

for (int i = xStart; i \< xStart + N; i++)  
 {  
 for (int j = yStart; j \< yStart + N; j++)  
 {  
 yield return Get(i, j);  
 }  
 }  
 }  
[/csharp]

## Solving the board

Now for the fun part. Let's solve the board using a basic recursive backtracking brute force attempt:

[csharp]  
public class Solver  
{  
 public static Board Solve(Board b)  
 {  
 var nextOpen = b.NextEmpty();

if (nextOpen == null)  
 {  
 return b;  
 }

var taken = b.UsedNumbersInSpace(nextOpen);

var available = b.TotalSpaceValues  
 .Except(taken)  
 .ToList();

if (available.Count == 0)  
 {  
 return null;  
 }

foreach (var possible in available)  
 {  
 var newBoard = b.Snapshot();

newBoard.Set(nextOpen, possible);

var next = Solve(newBoard);

if (next != null)  
 {  
 return next;  
 }  
 }

return null;  
 }  
}  
[/csharp]

Let's assume that `b.NextEmpty()` returns the first value from the `emptyList` backing collection in the board. Basically giving you a random empty value on each iteration.

That would totally work, but what happens when you move to a 4x4 board? Brute forcing it no longer really works. The runtime of a board is n^n, where n is the number of characters. So for a sudoku of size 3, thats a 9^9 runtime of 387420489. Shitty, but doable. But for 4x4 now you're at 16^16 which is 18446744073709551616. Holy moly, our algorithm isn't gonna work anymore!

This is where a [constraint based](http://en.wikipedia.org/wiki/Constraint_programming) approach would work. Instead of just doing things as part of a single cycle (get empty spot, find available slots, put in valid piece, repeat), what if when we put in a number we also make some basic decisions and try to minimize the search space.

1. If a cell group only has 1 open position, fill it. Continue to iterate through the board until the rule comes up false. 
2. After it comes up false, return the next open position who has the least amount of available items to choose from. I.e. if cell (1,1) has the possibility of being [1,2,3,4,5] and cell (5,4) has the possibility of being [1,2], return cell (5,4) as the next empty cell. This maximizes your failure rate and means you spend less time backtracking since your decision trees are more likely to fail sooner.

It's constraint based because the moment we pin a cell that only has 1 possibility, we may have changed other parts of the board. Maybe now other spaces _also_ only have one available space! We can keep going down this path until there is no more easy wins. So by choosing these values we've used some basic rules and logic to help us with our brute force search (which we still need to do when we are given too many options).

Given this, an easy way to tack this into the code above is to modify the `NextEmpty` function.

[csharp]  
public Location NextEmpty()  
{  
 if (\_emptySpaces.Count == 0)  
 {  
 return null;  
 }

var possibles = new Dictionary\<Location, List\<int\>\>();

foreach (var emptySpace in \_emptySpaces)  
 {  
 possibles[emptySpace] = TotalSpaceValues.Except(UsedNumbersInSpace(emptySpace)).ToList();

if (possibles[emptySpace].Count == 1)  
 {  
 Set(emptySpace, possibles[emptySpace].First());

return NextEmpty();  
 }  
 }

return possibles.MinBy(kvp =\> kvp.Value.Count()).Key;  
}  
[/csharp]

So what this code does is as you call for the next empty, it tries to constrain the board when it finds a primo spot to pin. Keeping in mind that at each solving iteration a full copy of the board is returned, its ok to mutate the board with this side effect. As you work through sudoku on each iteration, the possible questionable space to work through minimizes and you can now reasonably solve 4x4 boards pretty quickly!

This is really just a C# implementation of [Peter Norvig's](http://norvig.com/sudoku.html) sudoku solver, and if you'd like to see the full source (including the same tests that Peter Norvig used) check out my [github](https://github.com/devshorts/Playground/tree/master/Sudoko).

