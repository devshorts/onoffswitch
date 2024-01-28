---
layout: post
title: A daily programmer - nuts and bolts
date: 2014-01-20 08:00:13.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dailyprogrammer
- haskell
- IO
- monad
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560993964;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4316;}i:1;a:1:{s:2:"id";i:4262;}i:2;a:1:{s:2:"id";i:4327;}}}}

permalink: "/2014/01/20/daily-programmer/"
---
I've mentioned [r/dailyprogrammer](http://www.reddit.com/r/dailyprogrammer) in [previous posts](http://onoffswitch.net/?s=daily+programmer), since I think they are fun little problems to solve when I have time on my hands. They're also great problem sets to do when learning a new language.

This time around I decided to do an [easy one](http://www.reddit.com/r/dailyprogrammer/comments/1sob1e/121113_challenge_144_easy_nuts_bolts/) with haskell.

## Nuts and bolts problem description

The goal is stated as:

> You have just been hired at a local home improvement store to help compute the proper costs of inventory. The current prices are out of date and wrong; you have to figure out which items need to be re-labeled with the correct price.
> 
> You will be first given a list of item-names and their current price. You will then be given another list of the same item-names but with the correct price. You must then print a list of items that have changed, and by how much.

The formal inputs and outputs:

> Input Description  
> The first line of input will be an integer N, which is for the number of rows in each list. Each list has N-lines of two space-delimited strings: the first string will be the unique item name (without spaces), the second string will be the price (in whole-integer cents). The second list, following the same format, will have the same unique item-names, but with the correct price. Note that the lists may not be in the same order!  
> Output Description
> 
> For each item that has had its price changed, print a row with the item name and the price difference (in cents). Print the sign of the change (e.g. '+' for a growth in price, or '-' for a loss in price). Order does not matter for output.

And the sample input/output:

> Sample Input 1  
> 4  
> CarriageBolt 45  
> Eyebolt 50  
> Washer 120  
> Rivet 10  
> CarriageBolt 45  
> Eyebolt 45  
> Washer 140  
> Rivet 10
> 
> Sample Output 1  
> Eyebolt -5  
> Washer +20

## My haskell solution

And here is my haskell solution

```haskell
  
module Temp where

import Control.Monad  
import Data.List

data Item = Item { name :: String, price :: Integer }  
 deriving (Show, Read, Ord, Eq)

strToLine :: String -\> Item  
strToLine str = Item name (read price)  
 where  
 name:price:\_ = words str

formatPair :: (Item, Item) -\> [Char]  
formatPair (busted, actual) = format  
 where  
 diff = price actual - price busted  
 direction = if diff \> 0 then "+" else "-"  
 format = name busted ++ " " ++ direction ++ show (abs diff)

getPairs :: IO [(Item, Item)]  
getPairs = do  
 n \<- readLn  
 let readGroup = fmap (sort . map strToLine) (replicateM n getLine)  
 old \<- readGroup  
 new \<- readGroup  
 let busted = filter (\(a,b) -\> a /= b) $ zip old new  
 return $ busted

printPairs :: IO [(Item, Item)] -\> IO [String]  
printPairs pairs = fmap (map formatPair) pairs  

```

I had a lot of fun with this one, since it really forced me to understand and utilize `fmap` given that you had to deal with being in the `IO` monad. I also liked being "forced" to separate the IO from the pure. I say forced in quotes because it's really not that helpful to do all your work in the IO function; it's not reusable.

Also I found that by sticking to strongly typed data I had a more difficult time than if I had just leveraged the fact that the input was really a key value pair. However, the engineer in me knows that things could change, and I hate taking shortcuts. By strongly typing the input data and separating out the parsing function from the code that does filtering and formatting, we could extend the problem set to include other fields without having to jump back to the IO code.

Anyways, things are getting easier with haskell, but I'm still struggling with leveraging all the available libraries and constructs. I guess that just comes with time and practice.

