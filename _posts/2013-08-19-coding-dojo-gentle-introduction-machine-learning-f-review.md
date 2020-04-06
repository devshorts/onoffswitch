---
layout: post
title: 'Coding Dojo: a gentle introduction to Machine Learning with F# review'
date: 2013-08-19 00:56:12.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- F#
- machine learning
- meetup
meta:
  _su_rich_snippet_type: none
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558691217;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4126;}i:1;a:1:{s:2:"id";i:4275;}i:2;a:1:{s:2:"id";i:4209;}}}}

permalink: "/2013/08/19/coding-dojo-gentle-introduction-machine-learning-f-review/"
---
Recently I organized an [F# meetup in DC](http://www.meetup.com/F-meetup-in-Dupont-Circle/), and for our first event we brought in a wonderful speaker (Mathias Brandewinder) who's topic was called: "_Coding Dojo: a gentle introduction to Machine Learning with F#_".

I was certainly a little nervous about our first meetup, but a ton of great people came out: from experienced F# users, to people who had used other functional languages (like OCaml), to people with no functional experience. The goal of the meetup was to write a k-nearest neighbors classifier for a previously posted [kaggle](http://www.kaggle.com/c/digit-recognizer/data) exercise to classify pixellated numbers.

[caption width="600" align="alignnone"] ![](https://pbs.twimg.com/media/BR1-jWLCUAEO6E3.jpg) Mathias introducing F#[/caption]

Mathias did a great job of breaking people up into groups and then explaining what is machine learning and the criteria of the project in a surprsingly short time period. I think people were a little scared of jumping in since he only talked for about 10 to 15 minutes, but in place of a long lecture Mathias had a really well put together guided document that encouraged users to play and interact with F#.

The first step was to create an F# project and to download his fsx gist. The gist was broken down into 7 steps where each step walked a user through the basics of F# and machine learning to build their classifier. For example, one step was how to execute lines in F# interactive. Another step was explaining the map function. Another step talked about how to read a file and parse a csv. And yet another discussed distance functions and converting raw data into records.

[caption width="600" align="alignnone"] ![](https://pbs.twimg.com/media/BR0tUFACcAAJRre.jpg) The meetup group[/caption]

In the end, if you followed his steps, in a span of under 2 hours, even a novice could end up with a fully working classifier! The classifier's accuracy, by default, was about 94.4%. Not too bad.

I wanted to share my version of his classifer which is based off of Mathias' well guided steps.

[fsharp]  
open System  
open System.IO

type Number = { Label: string; Pixels: int[] }

let splitLine (line:String) = line.Split([|','|])

let extract file = File.ReadAllLines file |\> Array.map splitLine

let strippedHeaders (arr:'a[]) = arr.[1..]

let convertToInt (str:string) = Convert.ToInt32 str

let lineToInt arr = Array.map convertToInt arr

let linesAsInts = Array.map lineToInt

let toNum (line:int[]) = {Label = line.[0].ToString(); Pixels = line.[1..] }

let convertToNum lines = Array.map toNum lines

let dist (a:int) (b:int) = (a-b)\*(a-b)

let arrayDist = Array.map2 dist

let totalDist a b = arrayDist a b |\> Array.reduce (+)

let train file =  
 extract file  
 |\> strippedHeaders  
 |\> linesAsInts  
 |\> convertToNum

let kNNSet trainingSet pixels k =  
 trainingSet  
 |\> Array.map (fun i -\> (i.Label, totalDist i.Pixels pixels))  
 |\> Array.sortBy (fun (label, dist) -\> dist)  
 |\> fun sorted -\> sorted.[0..(k - 1)]

let classify trainingSet pixels k =  
 kNNSet trainingSet pixels k  
 |\> Array.toSeq  
 |\> Seq.groupBy (fun (label, dist) -\> label)  
 |\> Seq.maxBy (fun (label, items) -\> Seq.length items)  
 |\> fun (label, items) -\> label

let accuracy trainingSet validationSet k =  
 Array.map (fun i -\>  
 let result = classify trainingSet i.Pixels k  
 result = i.Label) validationSet  
 |\> Array.map(fun i -\> if i = true then 1 else 0)  
 |\> Array.sum  
 |\> fun sum -\> (double)sum / (double)(Array.length validationSet)  
 |\> fun acc -\> (int)(acc \* 100.0)

let training = train @"C:\Projects\Personal2\DcDojo\DcDojo\trainingsample.csv"  
let validation = train @"C:\Projects\Personal2\DcDojo\DcDojo\validationsample.csv"  
[/fsharp]

Had I written this without following his steps I probably would have inlined a lot of the simple helper functions, but I wanted to show how Mathias really brought the "_start small, build big_" mentality to the project. This is something that really works well in functional languages and I think all the meetup participants picked up on that.

Another meetup participant (my coworker Sam) [also posted](http://tech.blinemedical.com/machine-learning-with-f-and-c-side-by-side/) his kNN classifier, so go check it out and worked through it with a side by side C# example which was cool.

If you get a chance to see Mathias during his [summer of F# tour](http://www.clear-lines.com/blog/) you should! While DC was on the tail end of the trip, Boston and Detroit still are on the agenda.

--  
Edit:

Here is a youtube of a portion of the dojo:

<iframe width="420" height="315" src="//www.youtube.com/embed/MW_Km-vr1eE" frameborder="0" allowfullscreen></iframe>

