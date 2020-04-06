---
layout: post
title: K-Means Step by Step in F#
date: 2013-01-28 10:47:28.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- algorithms
- F#
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1051411444'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561012586;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4170;}i:1;a:1:{s:2:"id";i:3847;}i:2;a:1:{s:2:"id";i:1873;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/01/28/k-means-step-by-step-in-f/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/k-means-step-by-step-in-f/)_

Recently we had a discussion on clustering techniques at one of our weekly tech talks and the [k-means clustering](http://en.wikipedia.org/wiki/K-means_clustering) algorithm came up. K-means is considered one of the simplest unsupervised machine learning techniques and I thought it would be cool to try my hand at an F# implementation of it.

In short, k-means is a way to put data into groups based on distance between nearest neighbors. Technically it works with any dimension but for this post I'll stick with 1-d data to make things easy.

## K-means background

Imagine you have some 1-dimensional data like

[csharp]  
1, 2, 5, 14, 17, 19, 20  
[/csharp]

And you want to group this into two groups. Pretty easily you can see that 1, 2, and 5 go into one group, and 14, 17, 19, 20 should go in another group. But why? For this small data set we intuitively figured out that 1, 2, and 5 are close together, but at the same time far away from 14, 17, 19, and 20, which are also close together. Really what we did was we calculated the centroid of each group and assigned the values to the centroid. A centroid is a fancy way of saying center point of a group. It's calculated by taking the average of a group.

For example, if we have the group

[csharp]  
1, 2, 5  
[/csharp]

What is its centroid? The answer is

[csharp]  
(1 + 2 + 5) / 3 = 2.666  
[/csharp]

So with k-means you end up with k centroids and a bunch of data grouped to that centroid. It's grouped to that centroid because that data is closest to that centroid vs any other centroid. To calculate the centroids and grouping k-means uses an [iterative process](http://home.dei.polimi.it/matteucc/Clustering/tutorial_html/kmeans.html):

- Pick k random points. So if you are doing a k=2 clustering, just pick 2 random points from your data set. These are going to be our initial centroids. It doesn't matter if it's right, or even if the points are the same. The algorithm is going to fine tune things for us. Technically this is called the [Forgy Method](http://en.wikibooks.org/wiki/Data_Mining_Algorithms_In_R/Clustering/K-Means#Implementation) of initialization. There are a bunch of other ways to initialize your centroids, but Forgy seems to be pretty common.
- For every point, calculate its distance from the centroid. So lets say we picked points 1 and 2 as our centroids. The [distances](http://en.wikipedia.org/wiki/Euclidean_distance) then look like this:[csharp]  
 Δ1 Δ2  
1 0 1  
2 1 0  
5 4 3  
14 13 12  
17 16 15  
19 18 17  
20 19 18  
[/csharp]
- Now what we do is assign each data point to its nearest centroid. So, obviously, point 1 is closest to the initial arbitrary point 1 centroid. And you can see that basically everything else is closer to the arbitrary point 2 centroid. So we're left with a grouping like this now:[csharp]  
data around centroid 1: 1  
data around centroid 2: 2, 5, 14, 17, 19, 20  
[/csharp]
- Now, the next step is to calculate new centroids based on these groups. For this example, we can take the average of all the points together to find the new center. So the new centroids become 1 and 12.83. Now we go back to step 2 using the new centroids. If you keep track of the current centroid and the previous centroid, then you can stop the k-means calculation when centroids don't change between iterations. At this point everything has converged and you've got your k clusters

Unfortunately k-means doesn't always converge, so you can add a convergence delta (i.e. if the centroid between the current and previous iterations hasn't really changed much, so it's within some acceptable range) or an iteration limit (only try to cluster 100 times) so you can set bounds on the clustering.

## Implementation

The first thing to do for my implementation is to define a structure representing a single data point. I created a `DataPoint` class that takes a float list which represents the data points dimensions. So, a 1-dimesional data point would have a float list of one element. A 2-d data point has one of two points (x and y), etc. To give credit where credit is due, I took the dimensional array representation from [another F# implementation](http://www.navision-blog.de/2009/01/29/n-dimensional-k-means-clustering-with-f/).

[fsharp]  
type DataPoint(input:float list) =  
 member this.Data = input  
 member this.Dimensions = List.length input  
 override x.Equals(yobj) =  
 match yobj with  
 | :? DataPoint as y -\> (x.Data = y.Data)  
 | \_ -\> false  
 static member Distance (d1:DataPoint) (d2:DataPoint) =  
 if List.length d1.Data \<\> List.length d2.Data then  
 0.0  
 else  
 let sums = List.fold2(fun acc d1Item d2Item -\>  
 acc + (d2Item - d1Item)\*\*2.0  
 ) 0.0 d1.Data d2.Data  
 sqrt(sums)  
[/fsharp]

It has its data, the dimensions, an equality overload (so I can compare data points by their data and not by reference), as well as a static helper method to calculate the euclidean distance between two n-dimensional points. Here I'm using the `fold2` method which can fold two lists simultaneously.

## Alias data types

For fun, I wanted to use F#'s data aliasing feature, so I created some helper types to make dealing with tuples and other data pairing easier. I've defined the following

[fsharp]  
type Centroid = DataPoint

type Cluster = Centroid \* DataPoint List

type Distance = float

type DistanceAroundCentroid = DataPoint \* Centroid \* Distance

type Clusters = Cluster List

type ClustersAndOldCentroids = Clusters \* Centroid list  
[/fsharp]

The only unfortunate thing is you need to explicitly mention aliased types for the type inference to work properly. The reason is because F# will prefer native types to typed aliases (why choose `Distance` over float unless you are told to), but it would've been nice if it had some way to prefer local type aliases in a module.

## Centroid assignment

This next major step is to take the current raw data list and what the current centroids are and cluster them. A cluster is a centroid and its associated data points. To make things clearer, I created a helper function `distFromCentroid` which takes a point, a centroid, and returns a tuple of point \* centroid \* distance.

[fsharp]  
let private distFromCentroid pt centroid : DistanceAroundCentroid = (pt, centroid, (DataPoint.Distance centroid pt))  
[/fsharp]

The bulk of the k-means work is in `assignCentroids`. First, for every data point, we calculate its distance from the current centroid. Then we select the centroid which is closest to our data points and create a list of point \* centroid tuples. Once we have a list of point \* centroid tuples, we want to group this list by the second value in the tuple: the centroid. Here, I'm leveraging the pipe operator to do the grouping and mapping. First, we group the lists by the centroid. This gives us a list of `centroid * (datapoint * centroid)` items. But this isn't quite right yet. We want to end up with a list of `centroid * dataPoint list` so I passed this temporary list through some other minor filtering and formatting.

[fsharp]  
(\*  
 Takes the input list and the current group of centroids.  
 Calculates the distance of each point from each centroid  
 then assigns the data point to the centroid that is closest.  
\*)  
let private assignCentroids (rawDataList:DataPoint list) (currentCentroids:Clusters) =  
 let dataPointsWithCentroid =  
 rawDataList  
 |\> List.map(fun pt -\>  
 let currentPointByEachCentroid = List.map(fun (centroid, \_) -\> distFromCentroid pt centroid) currentCentroids  
 let (\_, nearestCentroid, \_) = List.minBy(fun (\_, \_, distance) -\> distance) currentPointByEachCentroid  
 (pt, nearestCentroid)  
 )

(\*  
 |\> Group all the data points by their centroid  
 |\> Select centroid \* dataPointList  
 |\> For each previous centroid, calculate a new centroid based on the aggregated list  
 \*)

List.toSeq dataPointsWithCentroid  
 |\> Seq.groupBy (fun (\_, centr) -\> centr)  
 |\> Seq.map(fun (centr, dataPointsWithCentroid) -\>  
 let dataPointList = Seq.toList (Seq.map(fun (pt, \_) -\> pt) dataPointsWithCentroid)  
 (centr, dataPointList)  
 )  
 |\> Seq.map(fun (cent, dataPointList) -\>  
 let newCent = calculateCentroidForPts dataPointList  
 (newCent, dataPointList)  
 )  
 |\> Seq.toList

[/fsharp]

## Calculating new centroids

Since we've grouped data points by centroid, we now need to calculate what the new centroid for that list is. This is where I employ a disclaimer. For my implementation of n-dimensional centroids I averaged each dimension together to get a new value. I'm not totally sure this is actually valid for arbitrary geometrical shapes (given by n-dimensions), but if it's not at least this is the only method that needs to be updated.

First I create an empty n-dimension array of all zeros, initialized by the dimensions of the first point in the list (all the dimensions should be the same so it doesn't matter which point we choose). This is the seed for list for the next fold. I'm not actually going to fold the value into a single element; I'm going to fold all indexes of the data point list together into a new array by adding each dimension together.

For example, if I have two points of two dimensions [1, 2] and [3, 4] I add 1 and 3 together to make the first dimension, then 2 and 4 together to make the second dimension. This gives me an intermediate vector of [4,6]. Then I can average each dimension by the number of data points used, so in our example we used two data points, so the new centroid would be [2, 3]. Since a centroid is the same as a data point (even though one that doesn't exist in our original raw set), we can return the new centroid as a data point. Having everything treated as data points makes the calculations easier.

[fsharp]  
(\*  
 take each float list representing an n-dimesional space for every data point  
 and average each dimension together. I.e. element 1 for each point gets averaged together  
 and element 2 gets averaged together. This new float list is the new nDimesional centroid  
\*)

let private calculateCentroidForPts (dataPointList:DataPoint List) =  
 let firstElem = List.head dataPointList  
 let nDimesionalEmptyList = List.init firstElem.Dimensions (fun i-\> 0.0)  
 let addedDimensions = List.fold(fun acc (dataPoint:DataPoint) -\> addListElements acc dataPoint.Data (+))  
 nDimesionalEmptyList  
 dataPointList

// scale the nDimensional sums by the size  
 let newCentroid = List.map(fun pt -\> pt/((float)(List.length dataPointList))) addedDimensions  
 new DataPoint(newCentroid)  
[/fsharp]

Here is the function to aggregate list elements together

[fsharp]  
(\* goes through two lists and adds each element together  
 i.e. index 1 with index 1, index 2 with index2  
 and returns a new list of the items aggregated  
\*)

let private addListElements list1 list2 operator =  
 let rec addListElements' list1 list2 accumulator operator =  
 if List.length list1 = 0 then  
 accumulator  
 else  
 let head1 = List.head list1  
 let head2 = List.head list2  
 addListElements' (List.tail list1) (List.tail list2) ((operator head1 head2)::accumulator) operator  
 addListElements' list1 list2 [] operator  
[/fsharp]

I'm using a hidden accumulator so that the function will be tail recursive.

## Cluster runner

The above code only does one single cluster iteration though . What we need is a clustering runner that does the clustering iterations. This runner will be responsible for determining when to stop the clustering. There are four functions here:

- `getClusters` and `getOldCentroids` are helper methods to pull data out of the tuples (fst and snd are built in functions)
- `cluster` is a basic method that will cluster as many times as necessary until the centroids stop changing
- `clusterWithIterationLimit` is the bulk method that takes in the raw data, the clustering amount (k), the max number of clustering iterations, and a convergence delta to avoid infinite loops

For each iteration create a new set of clusters, then test to see if if the centroids are the same. If they are, we've converged and we can end. If not, see if the clusters are "close enough" (the convergence delta). The final base case is if we've clustered the max iterations then return.

[fsharp]  
(\*  
 Start with the input source and the clustering amount  
 but also take in a max iteration limit as well as a converge  
 delta.

continue to cluster until the centroids stop changing or  
 the iteration limit is reached OR the converge delta is achieved

Returns a new "Clusters" type representing the centroid  
 and all the data points associated with it  
\*)

let private getClusters (cluster:ClustersAndOldCentroids) = fst cluster  
let private getOldCentroids (cluster:ClustersAndOldCentroids) = snd cluster

let clusterWithIterationLimit data k limit delta =  
 let initialClusters:Clusters = initialCentroids (List.toArray data) k  
 |\> Seq.toList  
 |\> List.map (fun i -\> (i,[]))

let rec cluster' (clustersWithOldCentroids:ClustersAndOldCentroids) count =  
 if count = 0 then  
 getClusters clustersWithOldCentroids  
 else  
 let newClusters = assignCentroids data (getClusters clustersWithOldCentroids)  
 let previousCentroids = getOldCentroids clustersWithOldCentroids  
 let newCentroids = extractCentroidsFromClusters newClusters

// clusters didnt change  
 if listsEqual newCentroids previousCentroids then  
 newClusters  
 else if centroidsDeltaMinimzed previousCentroids newCentroids delta then  
 newClusters  
 else  
 cluster' (newClusters, newCentroids) (count - 1)

cluster' (initialClusters, extractCentroidsFromClusters initialClusters) limit

(\*  
 Start with the input source and the clustering amount.  
 continue to cluster until the centroids stop changing  
 Returns a new "Clusters" type representing the centroid  
 and all the data points associated with it  
\*)

let cluster data k = clusterWithIterationLimit data k Int32.MaxValue 0.0  
[/fsharp]

## Demo

Using the sample data from the beginning of this post, let's run it through my k-means clusterer:

[fsharp]  
module KDataTest

open System  
open KMeans

let kClusterValue = 2

let sampleData : KMeans.DataPoint list = [new DataPoint([1.0]);  
 new DataPoint([2.0]);  
 new DataPoint([5.0]);  
 new DataPoint([14.0]);  
 new DataPoint([17.0]);  
 new DataPoint([19.0]);  
 new DataPoint([20.0])]

KMeans.cluster sampleData kClusterValue  
 |\> Seq.iter(KMeans.displayClusterInfo)

Console.ReadKey() |\> ignore

[/fsharp]

And the output is

[csharp]  
Centroid [2.66666666666667], with data points:  
[1], [2], [5]

Centroid [17.5], with data points:  
[14], [17], [19], [20]  
[/csharp]

Full source available at our [github](https://github.com/blinemedical/KMeans)

