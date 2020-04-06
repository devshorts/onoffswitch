---
layout: post
title: Linear separability and the boundary of wx+b
date: 2013-07-01 08:00:01.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Discussion
tags:
- machine learning
- neural networks
meta:
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561917902;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3615;}i:1;a:1:{s:2:"id";i:3942;}i:2;a:1:{s:2:"id";i:4170;}}}}

permalink: "/2013/07/01/meaning-wtxb/"
---
In machine learning, everyone talks about weights and activations, often in conjunction with a formula of the form `wx+b`. While reading [machine learning in action](http://www.manning.com/pharrington/) I frequently saw this formula but didn't really understand what it meant. Obviously its a line of some sort, but what does the line mean? Where does w come from? I was able to muddle past this for decision trees, and naive bayes, but when I got to support vector machines I was pretty confused. I wasn't able to follow the math and conceptually things got muddled.

At this point, I switched over to a different book, machine learning an algorithmic perspective.

[![41o11gP3WpL._BO2,204,203,200_PIsitb-sticker-arrow-click,TopRight,35,-76_AA300_SH20_OU01_.jpg](http://onoffswitch.net/wp-content/uploads/2013/06/41o11gP3WpL._BO2204203200_PIsitb-sticker-arrow-clickTopRight35-76_AA300_SH20_OU01_.jpg)](http://www.amazon.com/Machine-Learning-Algorithmic-Perspective-Recognition/dp/1420067184/ref=zg_bs_tab_pd_mg_2)

Here, the book starts with a discussion on neural networks, which are directly tied to the equation of `wx+b` and the concept of weights. I think this book is much better than the other one I was reading. The author does an excellent job of describing the big picture of what each algorithm is, followed by how the math is derived. This helped put things in perspective for me and let me peek into the workings of each algorithm without any glossed over magic.

## Neural networks

In a neural network, you take some set of inputs, multiply them by some value, and if the sum of all the inputs times that value is greater than some threshold (defined by a function) then the neuron fires. The thing that you multiply the input by is called the weight. The threshold is determined by an activation function.

[caption width="600" align="alignnone"] ![](http://onoffswitch.net/wp-content/uploads/2013/06/600px-ArtificialNeuronModel_english.png) Source image http://en.wikibooks.org[/caption]

But, lets say the inputs to all these nodes is zero, but you want the neuron to fire. Zero times any weights is zero so the node never can fire. This is why in neural networks you introduce a bias node. This bias node always has the value of one, but it also has its own weight. This bias node can offset inputs that are all zero that should trigger an activation.

[caption width="553" align="alignnone"] ![](http://onoffswitch.net/wp-content/uploads/2013/06/NN2.png) Source image http://www.codeproject.com/Articles/16419/AI-Neural-Network-for-beginners-Part-1-of-3[/caption]

A common way of calculating the activation of a neural network is to use matrix multiplication. Remembering that a neuron fires if the input times the weights is above some value, to find the sum you can take a row vector of the inputs and multiply it by a column vector of the weights. This gives you a single value that you can pass to a function that determines whether you want to fire or not.

![](http://onoffswitch.net/wp-content/uploads/2013/06/a2913ed385299a6e53da3f2e7c68d2fe.png)

In this way you can think of a neural network as a basic classifier. Given some input data it either fires or it doesn't. If the input data doesn't cause a firing then the class can be thought of 0, and if it does fire then the output class can be thought of as 1.

Now it makes sense why the **w** and the **x** are bold. Bold arguments, in math formulas, represent vectors and not scalars.

## Classifiers

But how does this line relate to classifiers? The point of classifiers is, given some input feature data, to determine what kind of thing it is. With a binary classifier, you want to find if the input data classifies as a one or a zero.

Here is an image representing the margin in a support vector machine

[caption width="220" align="alignnone"] ![](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2a/Svm_max_sep_hyperplane_with_margin.png/220px-Svm_max_sep_hyperplane_with_margin.png) Source wikipedia[/caption]

The points represent different instances, the color of the point tells you what class it is, and the x1 and x2 coordinates represent features. The point at (x1, x2) is an instance that maps to that feature set.

The line represents the point where if you took an x1 and x2 feature set, multiplied by and summed some fixed weights (so x1\*w1 + x2\*w2 + b), offset it by a determined bias, then everything above the line activates (neuron fires), and everything below the line doesn't activate. This means that the line is the delineation of what is one class vs what is another class.

What confused me here is that there is a discussion of w but the axis are only in relation to x1 and x2. Really what that means is that

[code]f(x1, x2) = w1\*x1+w2\*x2 + b[/code]

This is the ideal line that represents classification. **w** is a fixed vector here, the output function varies with the feature input. The b of the function represents the weight of the bias node: it's independent of the input data.

**wx** +b defined a linearly separable classifier. In two dimensional space this is a line, but you can also classify in any other space. If your input has 3 features you can create a separating plane, and in more dimensions you can create a hyperplane. This is the basis of kernel functions (mapping feature sets that aren't linearly separable in one space into a feature set that is linearly separable in another space: ie projection).

