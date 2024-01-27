---
layout: post
title: Building an ID3 decision tree
date: 2013-06-03 08:00:02.000000000 -07:00
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
- machine learning
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560643978;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4275;}i:1;a:1:{s:2:"id";i:4126;}i:2;a:1:{s:2:"id";i:3656;}}}}

permalink: "/2013/06/03/building-decision-tree/"
---
After following [Mathias Brandewinder's](http://clear-lines.com/blog/post/Decision-Tree-classification.aspx) series on converting the python from ["Machine Learning in Action"](http://www.manning.com/pharrington/) to F#, I decided I'd give the book a try myself. Brandewinder's blog is great and he went through chapter by chapter working through F# conversions. If you followed his series, this won't be anything new. Still, I decided to do the same thing as a way to solidify the concepts for myself, and in order to differentiate my posts I am reworking the python code into C#. For the impatient, the full source is available at my [github](https://github.com/devshorts/Playground/tree/master/MachineLearning/DecisionTree).

This post will discuss the ID3 decision tree algorithm. ID3 is an algorithm that's used to create a decision tree from a sample data set. Once you have the tree, you can then follow the branches of the tree until you reach a leaf and that will give you a classification for your sample.

For example, lets say you have a dataset like this. Each row represents some animal and its characteristics.

[![2013-05-25 11_57_19-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-11_57_19-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-11_57_19-MarkdownPad-2.png)

The data set clearly tells us what is and isn't a fish. Using ID3 you can take all your known sample data and build out a tree that would look like this in the end:

```
  
can survive without surfacing: yes  
 has flippers: yes  
 is fish: yes  
 has flippers: no  
 is fish: no  
can survive without surfacing: no  
 is fish: no  

```

If we get a piece of data that has features, but no class, to find the class all we have to do is follow the branches till we get a leaf which gives us the final class classifier.

With our sample, we can say _Can it survive without surfacing? If yes, then does it have flippers? If no, then its not a fish_. The nice thing about a decision tree like this, is that it can collapse common features for you. Notice how if your animal can't survive without surfacing, you don't even have to ask if it has flippers. It will never be a fish.

## The Data Set

The data set is what you'll use to build the original tree and it just contains a set of instances. An instance is like what I showed above, it's one piece of data in the set. Each instance also has features (also called axis). A feature would be "Has flippers", or "Can survive without surfacing". The final output of an instance is its class.

```csharp
  
public class DecisionTreeSet  
{  
 public List\<Instance\> Instances { get; set; }  
}  

```

```csharp
  
public class Instance  
{  
 public List\<Feature\> Features { get; set; }

public Output Output { get; set; }  
}  

```

```csharp
  
public class Feature  
{  
 public string Value { get; set; }  
 public string Axis { get; set; }

public Feature(string value, string axis)  
 {  
 Value = value;  
 Axis = axis;  
 }  
}  

```

## Splitting Features

Let's imagine we were to generate a decision without a decision tree. We have some input data set and we have the thing we want to classify. If the thing we are trying to classify matches an existing item in the set (all the feature values match), then that's easy. But, if the thing we are trying to classify has a mix of features that doesn't match any one item we can use an algorithm like this:

1. Find all instances in the data set that have the same value of a feature. Discard all other instances (as they won't match)
2. For the instances you found, remove the feature from their lists. We don't need it anymore, we already matched on it.
3. If we have only one instance left, or all the instances have the same output class, then we're that class. Otherwise go back to step 1 for the next feature

But doing this each time we want to classify an item is going to be expensive. By precomputing this information, and structuring it in a smart way, we can vastly improve the classification time of a new instance.

Like mentioned in step's 1 and 2, splitting the data set on a feature means returning a new set of instances from the original set who have the target feature value, and then removing that axis from the feature list. This will come in handy later when we try and determine which feature to create a branch in the decision tree from. Let me show an example using the original data set:

[![2013-05-25 11_57_19-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-11_57_19-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-11_57_19-MarkdownPad-2.png)

If I split on "Can survive without surfacing" with the value of "yes" I'd get:

[![split](http://onoffswitch.net/wp-content/uploads/2013/05/split1.jpg)](http://onoffswitch.net/wp-content/uploads/2013/05/split1.jpg)

So I've returned only the instances that had the value of "yes" in that column, AND I've removed the axis (column) and created a new sub data set.

Here is how I split the sets in C#

```csharp
  
public class DecisionTreeSet  
{  
 //...  
 public DecisionTreeSet Split(Feature feature)  
 {  
 return Split(feature.Axis, feature.Value);  
 }

public DecisionTreeSet Split(string axis, string value)  
 {  
 return new DecisionTreeSet  
 {  
 Instances = Instances.Select(i =\> i.Split(axis, value))  
 .Where(i =\> i.Features.Any())  
 .ToList()  
 };  
 }  
}  

```

```csharp
  
public class Instance  
{  
 //...

public Instance Split(string axis, string value)  
 {  
 var featureSplit = Features.Where(feature =\> !feature.IsMatch(axis, value)).ToList();

// no split happened  
 if (featureSplit.Count == Features.Count)  
 {  
 featureSplit = new List\<Feature\>();  
 }

return new Instance  
 {  
 Output = Output,  
 Features = featureSplit  
 };  
 }  
}  

```

## Measuring Order

While we can split the tree up and decide without a tree, we still have to wisely decide how to structure the tree. The goal of the tree building is to figure out what is the best way to structure the data. The question we're trying to answer is what feature makes the best branch in order to maximize the information in the tree. In other words, when would you choose to make a branch on "has flippers", vs "can survive without surfacing?". To figure this out we need a way to measure if we gained information by choosing one branch over the other.

How do you do that though? Well, we can measure the amount of disorder in a set using the [shannon entropy](http://en.wikipedia.org/wiki/Entropy_(information_theory)) formula. Shannon gives you a measurement of how mixed up the data is. The less mixed up the data you have, the better of a decision you are making on that particular feature. We're interested not in the disorder of the features, but only in the outputs of a set.

Calculating the entropy is pretty easy using shannons formula:

![](http://onoffswitch.net/wp-content/uploads/2013/05/efdf8c905c0f9dfd78002df6f20edb5d.png)

```csharp
  
public static double Entropy(DecisionTreeSet set)  
{  
 var total = set.Instances.Count();

var outputs = set.Instances.Select(i =\> i.Output).GroupBy(f =\> f.Value).ToList();

var entropy = 0.0;

foreach (var target in outputs)  
 {  
 var probability = (float)target.Count()/total;  
 entropy -= probability\*Math.Log(probability, 2);  
 }

return entropy;  
}  

```

For each possible output class type (fish/not fish), determine the probability of finding that class type in the total set. The sum of all those probabilities in the set is your entropy.

## Selecting the best axis to split on

Now that we have a way to create sub data sets, and a way to measure how well ordered those subsets are, we can measure if these data sets helps us predict the outcome better vs any other split. Doing a split for each feature's unique values will tell us which feature to split the set on.

This should make sense, since we want the resulting data sets after a split to have the least amount of entropy (more quickly converging to an answer). We need to test a split on each unique value of a feature (eg. yes:has flippers, and no:has flippers). The sum of the entropy of those splits will give us the measure of order of the next level deep in the tree.

First we figure out what the entropy of the total base set is. Then get all the unique values for a feature. In our sample set, "can survive without surfacing" has two unique values: "yes" and "no". Same with "has flippers". By summing the entropy for each split of a feature's unique values we can get the total entropy of the tree for that split. A large info gain tells us that splitting on an axis (the sum of entropy of splitting on all a features unique values) gave us a more homogenous and uniform final class output. A low info gain tells us that the final class output was still pretty mixed up and not so good.

```csharp
  
public static string SelectBestAxis(DecisionTreeSet set)  
{  
 var baseEntropy = Entropy(set);

var bestInfoGain = 0.0;

var uniqueFeaturesByAxis = set.UniqueFeatures().GroupBy(i =\> i.Axis).ToList();

string bestAxisSplit = uniqueFeaturesByAxis.First().Key;

foreach (var axis in uniqueFeaturesByAxis)  
 {  
 // calculate the total entropy based on splitting by this axis. The total entropy  
 // is the sum of the entropy of each branch that would be created by this split

var newEntropy = EntropyForSplitBranches(set, axis.ToList());

var infoGain = baseEntropy - newEntropy;

if (infoGain \> bestInfoGain)  
 {  
 bestInfoGain = infoGain;

bestAxisSplit = axis.Key;  
 }  
 }

return bestAxisSplit;  
}

private static double EntropyForSplitBranches(DecisionTreeSet set, IEnumerable\<Feature\> allPossibleAxisValues)  
{  
 return (from possibleValue in allPossibleAxisValues  
 select set.Split(possibleValue) into subset  
 let prob = (float) subset.NumberOfInstances/set.NumberOfInstances  
 select prob\*Entropy(subset)).Sum();  
}  

```

## Split the samples example

Lets trace through it with the sample set above. The original sets base entropy is 0.97095057690894.

[![2013-05-25 13_18_36-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_36-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_36-MarkdownPad-2.png)

[![2013-05-25 13_18_29-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_29-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_29-MarkdownPad-2.png)

```
  
The entropy for axis "can survive without surfacing" value "yes" is 0.550977512949583  
The entropy for axis "can survive without surfacing" value "no" is 0  

```

This makes sense, if we split on "no", then both of the "Is Fish" outputs are "no", so the class output is uniform. This is best kind of entropy!

Now, the other splits

[![2013-05-25 13_18_47-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_47-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_47-MarkdownPad-2.png)

[![2013-05-25 13_18_41-MarkdownPad 2](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_41-MarkdownPad-2.png)](http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-25-13_18_41-MarkdownPad-2.png)

```
  
The entropy for axis "has flippers" value "yes" is 0.800000011920929  
The entropy for axis "has flippers" value "no" is 0  

```

So while splitting on "has flippers" with the value of "no" gives us a zero entropy, splitting on the value "yes" gives a higher entropy (0.8) than splitting on "can survive without surfacing" with a value of "yes" (0.5509). In this case, splitting on "can survive" is a better split.

When all is said and done we can figure out which axis is the best to split on. After splitting on an axis, each potential feature value (yes, or no), becomes a separate branch in the decision tree. Each branch on a tree represents a new subset, and so you can recursively split the sets until the set only has on instance in it, OR, all the classes in all the instances are the same.

## Building the tree

Once you have the best axis to split on, building the tree comes together easily.

A tree is just a leaf, or some branches

```csharp
  
public class Tree  
{  
 public Output Leaf { get; set; }

public Dictionary\<Feature, Tree\> Branches { get; set; }  
}  

```

And to build the tree, recurse and build until all instances are of the same class, or all instances only have one feature left

```csharp
  
public class DecisionTreeSet  
{  
 //...  
 public Tree BuildTree()  
 {  
 if (InstancesAreSameClass || Instances.All(f =\> f.Features.Count() == 1))  
 {  
 return LeafTreeForRemainingFeatures();  
 }

var best = Decider.SelectBestAxis(this);

return SplitByAxis(best);  
 }

private Tree SplitByAxis(string axis)  
 {  
 if (axis == null)  
 {  
 return null;  
 }

// split the set on each unique feature value where the feature is  
 // if of the right axis  
 var splits = (from feature in UniqueFeatures().Where(a =\> a.Axis == axis)  
 select new {splitFeature = feature, set = Split(feature)}).ToList();

var branches = new Dictionary\<Feature, Tree\>();

// for each split, either recursively create a new tree  
 // or split the final feature outputs into leaf trees  
 foreach (var item in splits)  
 {  
 branches[item.splitFeature] = item.set.BuildTree();  
 }

return new Tree  
 {  
 Branches = branches  
 };  
 }

private Tree LeafTreeForRemainingFeatures()  
 {  
 if (InstancesAreSameClass)  
 {  
 return GroupByClass();  
 }

if (Instances.All(f =\> f.Features.Count() == 1))  
 {  
 return LeafForEachFeature();  
 }

return null;  
 }

private Tree LeafForEachFeature()  
 {  
 // each feature is the last item  
 var branches = new Dictionary\<Feature, Tree\>();

foreach (var instance in Instances)  
 {  
 foreach (var feature in instance.Features)  
 {  
 if (branches.Any(k =\> k.Key.Value == feature.Value))  
 {  
 continue;  
 }

branches[feature] = new Tree  
 {  
 Leaf = instance.Output  
 };  
 }  
 }

return new Tree  
 {  
 Branches = branches  
 };  
 }

private Tree GroupByClass()  
 {  
 var groupings = Instances.DistinctBy(i =\> i.Output.Value)  
 .ToDictionary(i =\> i.Features.First(), j =\> new Tree  
 {  
 Leaf = j.Output  
 });

if (groupings.Count() \> 1)  
 {  
 return new Tree  
 {  
 Branches = groupings  
 };  
 }

return new Tree  
 {  
 Leaf = groupings.First().Value.Leaf  
 };  
 }

public IEnumerable\<Feature\> UniqueFeatures()  
 {  
 return Instances.SelectMany(f =\> f.Features).DistinctBy(f =\> f.Axis + f.Value).ToList();  
 }  
}  

```

## Testing it

The examples I've used all are all straight from the Machine Learning in Action book, but we can try on a more realistic data set. The [UCI Machine Repository](http://archive.ics.uci.edu/ml/datasets.html) has a bunch of great test data sets to use.

Using my decision tree, I've built a classifier using the [car evaluation](http://archive.ics.uci.edu/ml/datasets/Car+Evaluation) data set.

First, lets parse the data set. I made a generic line reader class that can handle emitting data sets for static files

```csharp
  
public abstract class LineReader : IParser  
{  
 public DecisionTreeSet Parse(string file)  
 {  
 var set = new DecisionTreeSet();

set.Instances = new List\<Instance\>();

using (var stream = new StreamReader(file))  
 {  
 while (!stream.EndOfStream)  
 {  
 var line = stream.ReadLine();  
 set.Instances.Add(ParseLine(line));  
 }  
 }

return set;  
 }

protected abstract Instance ParseLine(string line);  
}  

```

And implement the line parser for this set

```csharp
  
public class Car : LineReader  
{  
 protected override Instance ParseLine(string line)  
 {  
 var splits = line.Split(new[] { ',' }, StringSplitOptions.RemoveEmptyEntries);

var buying = splits[0];  
 var maintence = splits[1];  
 var doors = splits[2];  
 var people = splits[3];  
 var lugBoot = splits[4];  
 var safety = splits[5];

return new Instance  
 {  
 Output = new Output(splits[6], "car acceptability"),  
 Features = new List\<Feature\>  
 {  
 new Feature(buying, "buying"),  
 new Feature(maintence, "maintence"),  
 new Feature(doors, "doors"),  
 new Feature(people, "people"),  
 new Feature(lugBoot, "lugboot"),  
 new Feature(safety, "safety")  
 }  
 };  
 }  
}  

```

Finally to build the tree

```csharp
  
[Test]  
public void TestCar()  
{  
 var file = @"..\..\..\Assets\CarEvaluation\car.data";  
 var set = new Car().Parse(file);

var tree = set.BuildTree();

tree.DisplayTree();

foreach (var instance in set.Instances)  
 {  
 Assert.That(Tree.ProcessInstance(tree, instance).Value, Is.EqualTo(instance.Output.Value));  
 }  
}  

```

Which gives us a pretty big tree. `Tree.ProcessInstance` takes a tree and a sample instance and returns you the class. The test runs through all the sample data and validates that the tree returns the appropriate output.

Feel free to [check out the code](https://github.com/devshorts/Playground/tree/master/MachineLearning/DecisionTree) and run it to see the actual tree

## Notes

When you use a decision tree you don't want to have to rebuild the tree each time you use it, so it's nice that the tree is easily serialzable. If you train the tree on a large data set you can store it and deserialize it whenever you need to use it.

