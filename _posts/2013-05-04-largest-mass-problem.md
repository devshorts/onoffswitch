---
layout: post
title: The largest mass problem
date: 2013-05-04 18:05:45.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- algorithms
- F#
- flood fill
- tail recursion
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559119788;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4411;}i:1;a:1:{s:2:"id";i:3899;}i:2;a:1:{s:2:"id";i:1043;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/04/largest-mass-problem/"
---
I was recently asked to write some code to find the largest contiguous group of synonymous elements in a two dimensional array. The idea is that you want to find the largest "land mass" in a problem where you have a game board that looks something like

[code]  
L L L W  
W L L W  
L W W W  
W L L W  
[/code]

Where `L` stands for land, and `W` stands for water. In this example, the largest land mass would be of size 5. But there are also 2 other land masses, one of size one, and another of size two. Elements can be contiguous only if their direct adjacent neighbor is the same type, so diagonals don't count.

In general, you can think of the largest mass problem as almost exactly the same as the [flood fill problem](http://en.wikipedia.org/wiki/Flood_fill) in image graphics. Except with flood fill, you are given a location and you want to find all the contiguous areas to fill. Here, you don't know where to start, you have to find all the contiguous areas in the board.

To me, this solution smells of recursion. You need a way to start at a point, and branch in all directions following until you find non land areas. This way you can go up, down, left and right starting at a point and each direction will return to you what it found.

## The board

First, lets define our example board:

[fsharp]  
type Earth =  
 | Land  
 | Water

let board = array2D [[Land; Land; Land; Water;];  
 [Water; Land; Land; Water;];  
 [Land; Water; Water; Water;];  
 [Water; Land; Land; Water;]]  
[/fsharp]

## Moving around

Next, lets define some helper methods. Since I know I'm going to have to branch up, down, left and right. I also know that I need to cover edge conditions such as when I'm iterating over the board and I am going to step off the board edge (beyond the size of the 2d array). I'm treating the current position on the board as an integer tuple representing x and y.

[fsharp]  
let moveRight position =  
 let (x,y) = position  
 (x + 1, y)

let moveLeft position =  
 let (x,y) = position  
 (x - 1, y)

let moveUp position =  
 let (x,y) = position  
 (x, y + 1)

let moveDown position =  
 let (x,y) = position  
 (x, y - 1)

let xSize board = Array2D.length1 board

let ySize board = Array2D.length2 board

let offBoard position board =  
 let (x,y) = position  
 x \< 0 || y \< 0 || x \>= (xSize board) || y \>= (ySize board)  
[/fsharp]

## Keeping track of where you've been

I also know that since I'm going to be branching through this board in different recursive iterations, I need to be able to keep track of cells that I've already worked on. This makes sure that one branch (for example going left) doesn't re-process cells that were processed by another branch (like one that went up). I have two methods here, one to just cons the current position to the previous positions list, and another to help me find if the current position is in a positions list.

[fsharp]  
let markPosition position previousSpots = position::previousSpots

let positionExists position list =  
 List.exists(fun pos -\> pos = position) list  
[/fsharp]

## Did I find one?

Also, I can create a helper method that tells me if the current position I'm on matches the target type that I want.

[fsharp]  
let positionOnTarget position board target =  
 if offBoard position board then  
 false  
 else  
 let (x, y) = position  
 (Array2D.get board x y) = target  
[/fsharp]

You may have noticed that a lot of these helper functions are only one line, and sometimes just wrap another one line built in F# functionality. I like to do it that way for readability sake.

## Finding masses

Lets start with what flood fill does. Given a position, find all the contiguous elements. Each time we find a block it returns the block positions it found and the elements it already processed as a tuple

[fsharp]  
type Board\<'T\> = 'T[,]

type X = int

type Y = int

type Position = X \* Y

type PositionList = Position list

type ProcessedPositions = PositionList

type ContiguousPoints = PositionList

type MassFinder = ContiguousPoints \* ProcessedPositions

(\*  
 Looks for a specified contigoius block  
 and keeps track of processed positions using a  
 reference cell of a list of positions (supplied by the caller)  
\*)

let findMassStartingAt (position:Position) (board:Board\<'A\>) (target:'A) (positionSeed:ProcessedPositions) : MassFinder =  
 let rec findMassStartingAt' position (currentMass:ContiguousPoints, processedList:ProcessedPositions) =

// if you move off the board return  
 if offBoard position board then  
 (currentMass, processedList)

// if you already processed this position then don't do anything  
 else if positionExists position processedList then  
 (currentMass, processedList)  
 else

// branch out left, up, right, and down and see what you can find  
 let up = moveUp position  
 let down = moveDown position  
 let left = moveLeft position  
 let right = moveRight position

let found = positionOnTarget position board target

match found with  
 | true -\>  
 (position::currentMass, position::processedList)  
 |\> findMassStartingAt' up  
 |\> findMassStartingAt' down  
 |\> findMassStartingAt' left  
 |\> findMassStartingAt' right

| false -\>  
 // if you didn't find anything return the masses that you  
 // found prevoiusly  
 (currentMass, processedList)

findMassStartingAt' position ([], positionSeed)  
[/fsharp]

Each time the mass function is called it returns the masses it found. This is why up, down, left and right are all being piped a new list telling it what's already been found. By the end of the entire search the recursive calls have returned all available contiguous blocks starting from the original seed position. Also instead of passing the board to the inner list I'm leveraging the parent closure to reference the board.

But, this only finds a mass if we told it where to start. To search for other masses I opted to brute force the problem and iterate over the entire 2d array, re-using the function that knew how to find a single mass. To iterate over the 2d array I created the following function

[fsharp]  
(\*  
 Iterate over each element in a 2d array, passing the x and y  
 coordinate and the board, to the supplied function  
 which can return an item. The items are all cons together  
 and the function returns a new list  
\*)

let forEachElement (applier:(X -\> Y -\> Board\<'a\> -\> 'b)) (twoDimArray:Board\<'a\>) =  
 let mutable items = []  
 for x in 0..(xSize board) do  
 for y in 0..(ySize board) do  
 items \<- (applier x y twoDimArray)::items  
 items  
[/fsharp]

Which lets you apply a function to each element and return a new item. The other Array2D built in functions always created other 2D arrays, but I basically wanted to create a list based on the indexes and not just the element at those indexes.

Now our final contiguous searcher looks like this. Remember that each block we find returns a `MassFinder` tuple which is `ContiguousPoints * ProcessedPositions` so I am just picking out the contiguous blocks with the `fst` map.

[fsharp]  
(\*  
 Finds all contiguous blocks of the specified type  
 and returns a list of lists (each list is the points for a specific  
 block)  
\*)

let getContiguousBlocks board target =

// go through each board element and find masses starting at the  
 // the current position  
 // filter out any positions that found no masses  
 let findMass x y board = findMassStartingAt (x, y) board target []

forEachElement findMass board  
 |\> List.map fst  
 |\> List.filter (List.isEmpty \>\> not)  
[/fsharp]

Breaking things up like this also lets us solve the correlary problem of flood fill! I'm passing it an empty list as a seed to say we haven't processed any elements

[fsharp]  
(\*  
 Returns a list of points representing a contigious block  
 of the type that the point was at.  
\*)

let floodFillArea (point:Position) (canvas:Board\<'T\>) =  
 let (x, y) = point  
 let itemAtPoint = Array2D.get canvas x y

findMassStartingAt point canvas itemAtPoint [] |\> fst

[/fsharp]

## The test

Well, lets try it:

[fsharp]  
(\*  
 Test functions to run it  
\*)

let masses = getContiguousBlocks board Land

let largestList = List.maxBy(List.length) masses

let massAt = floodFillArea (2, 2) boardInt

let sizeOfMassAt22 = List.length massAt

System.Console.WriteLine("Largest mass is " + (List.length largestList).ToString());  
System.Console.WriteLine("Mass size at (2,2) is " + sizeOfMassAt22.ToString());  
[/fsharp]

And the answer is...

[code]  
Largest mass is 5  
Mass size at (2,2) is 6  
[/code]

Where index (2,2) was the large water block in the middle.

Looks like it worked

## But what about stack depth?

Now that there is a basic working version, we can make this a bit more advanced. If we use continuation passing style then we can make our multiple branching recursion tail recursive. Here is a rewritten version of the mass finder function but which uses continuations:

[fsharp]  
let findMassStartingAt (position:Position) (board:Board\<'A\>) (target:'A) (positionSeed:ProcessedPositions) : MassFinder =  
 let rec findMassStartingAt' position (currentMass:ContiguousPoints, processedList:ProcessedPositions) cont =

// if you move off the board return  
 if offBoard position board then  
 cont((currentMass, processedList))

// if you already processed this position then don't do anything  
 else if positionExists position processedList then  
 cont((currentMass, processedList))  
 else

// branch out left, up, right, and down and see what you can find  
 let up = moveUp position  
 let down = moveDown position  
 let left = moveLeft position  
 let right = moveRight position

let found = positionOnTarget position board target

// track that we processed this element even if we don't find anything  
 let updatedProcess = position::processedList

match found with  
 | true -\>  
 let massState = (position::currentMass, updatedProcess)

findMassStartingAt' up massState (fun foundMassUp -\>  
 findMassStartingAt' down foundMassUp (fun foundMassDown -\>  
 findMassStartingAt' left foundMassDown (fun foundMassLeft -\>  
 findMassStartingAt' right foundMassLeft cont)))

| false -\>  
 // if you didn't find anything return the masses that you  
 // found previously  
 cont((currentMass, updatedProcess))

findMassStartingAt' position ([], positionSeed) id  
[/fsharp]

Instead of letting all the recursion bubble and piping that value to the next recursion, now we're capturing what to do when the next recursion is ready to run. By using the closure state we can capture what is the next point to go to, and we know that the next `MassFinder` that was previously processed will be passed to the continuation at each round. Now there's no worry about stack depth!

If we look at the IL that was generated for the inner recursive function we can really illustrate the point, that the F# compiler has emitted a [tail call opcode](http://stackoverflow.com/questions/15864670/generate-tail-call-opcode):

[csharp highlight="33"]  
.method assembly static  
 !!a 'findMassStartingAt\'@176'\<A, a\> (  
 !!A[0..., 0...] board,  
 !!A target,  
 int32 position\_0,  
 int32 position\_1,  
 class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\> currentMass,  
 class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\> processedList,  
 class [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2\<class [mscorlib]System.Tuple`2\<class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>, class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>\>, !!a\> cont  
 ) cil managed  
{  
 // Method begins at RVA 0x2218  
 // Code size 197 (0xc5)  
 .maxstack 16  
 .locals init (  
 [0] class [mscorlib]System.Tuple`2\<int32, int32\> position,  
 [1] class [mscorlib]System.Tuple`2\<int32, int32\> up,  
 [2] class [mscorlib]System.Tuple`2\<int32, int32\> down,  
 [3] class [mscorlib]System.Tuple`2\<int32, int32\> left,  
 [4] class [mscorlib]System.Tuple`2\<int32, int32\> right  
 )

// loop start  
 IL\_0000: ldarg.2  
 // .. removed ...  
 IL\_00ad: br IL\_0000  
 // end loop

IL\_00b2: ldarg.s cont  
 IL\_00b4: ldarg.s currentMass  
 IL\_00b6: ldarg.s processedList  
 IL\_00b8: newobj instance void class [mscorlib]System.Tuple`2\<class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>, class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>\>::.ctor(!0, !1)  
 IL\_00bd: tail.  
 IL\_00bf: callvirt instance !1 class [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2\<class [mscorlib]System.Tuple`2\<class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>, class [FSharp.Core]Microsoft.FSharp.Collections.FSharpList`1\<class [mscorlib]System.Tuple`2\<int32, int32\>\>\>, !!a\>::Invoke(!0)  
 IL\_00c4: ret  
} // end of method Print::'findMassStartingAt\'@176'  
[/csharp]

## Choose only non processed elements

I couldn't help myself so I revisited the code a little. I didn't want to brute force the entire board, instead I want to selectively choose the first unprocessed position to see if its got a contiguous block of what I want.

The basic idea is to generate an array reprenseting all of the boards positions. Then find the intersection of the points you've processed vs the available points. Then find the first point NOT in the intersection. If the resulting list is empty we've processed everything. If we've never processed anything just start from the top left corner.

You could cut out even more work if you cached the creation of the board tuple array elsewhere.

[fsharp]  
(\*  
 Finds all items of list2 that are not in list1  
\*)

let except list1 list2 =  
 let listContainsElement item = List.exists (fun i -\> i = item) list1  
 List.filter(fun item -\> not (listContainsElement item)) list2

(\*  
 Find first non processed position  
\*)

let firstNonProcessedPosition processedList xCount yCount =  
 match processedList with  
 | [] -\>  
 Some((0, 0))  
 | \_ -\>  
 if List.length processedList = (xCount \* yCount) then  
 None  
 else

// get an array representing (x, y) tuples of the entire board  
 let totalPositions = [0..xCount] |\> List.collect (fun x -\> [0..yCount] |\> List.map (fun y -\> (x, y)))

// set intersections from the total positions array and the entire board  
 let intersections = Set.intersect (Set.ofList totalPositions) (Set.ofList processedList)  
 |\> List.ofSeq

// exclude the intersections from the total list  
 let excludes = except intersections totalPositions

match excludes with  
 | [] -\> None  
 | \_ -\> Some(List.head excludes)  
[/fsharp]

And now we just need to use this new information to feed to the fill function to find our contiguous block of elements. This function is a little more complicated, but not by much.

[fsharp]  
(\*  
 Finds all contiguous blocks of the specified type  
 and returns a list of lists (each list is the points for a specific  
 block)  
\*)

let getContiguousBlocks board target =

let xCount = (xSize board) - 1  
 let yCount = (ySize board) - 1

let rec findBlocks' (blocks, processed:PositionList) =

let findMass x y board = findMassStartingAt (x, y) board target processed

// find the first non processed block  
 // and try and find its contigoius area  
 // if it isn't a valid area the block it returns will be  
 // empty and we can exclude it  
 match firstNonProcessedPosition processed xCount yCount with  
 | None -\> blocks  
 | Some (x, y) -\>  
 let (block, processed) = findMass x y board

findBlocks' ((match block with  
 | [] -\> blocks  
 | \_ -\> block::blocks), processed)

findBlocks' ([],[])  
[/fsharp]

Now, we keep processing until we've processed everyone. At that point, return what we found!

## View the full snippet

I'm posting the full snippet at [fsharp snippets](http://fssnip.net/ik)

