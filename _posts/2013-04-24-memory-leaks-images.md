---
layout: post
title: Images, memory leaks, GDI+, and the aggregate function
date: 2013-04-24 00:55:42.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- c#
- Image
- memory leak
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561976764;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3524;}i:1;a:1:{s:2:"id";i:4028;}i:2;a:1:{s:2:"id";i:3232;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/04/24/memory-leaks-images/"
---
I ran into a neat C# memory leak today that I wanted to share. It's not often you get a clear undeniable leak in C# and so I really had fun figuring this one out.

Look at this and see if you can spot the leak:

[csharp]  
public static class Extensions  
{  
 public static Image Append(this Image source, Image append)  
 {  
 var newImage = new Bitmap(source.Width + append.Width, source.Height);  
 using (var g = Graphics.FromImage(newImage))  
 {  
 g.DrawImage(source, 0, 0);

g.DrawImage(append, source.Width, 0);  
 }

return newImage;  
 }  
}

private static void Main(string[] args)  
{  
 var src = @"C:\users\anton\desktop\bigImage.jpg";

var images = Enumerable.Repeat(Image.FromFile(src), 25).ToList();

var appendedImage = images.Aggregate((acc, i) =\> acc.Append(i));

foreach (var image in images)  
 {  
 image.Dispose();  
 }

appendedImage.Dispose();

Console.ReadLine();  
}  
[/csharp]

What this code does is create 25 instances of my bigImage.jpg (6.5MB), and then creates a new image consisting of those 25 images side by side. The aggregate function folds the list into a single image using the `Append` extension method. This way the accumulator is the new running image.

Since `Image` is disposable, I've disposed of the source images, as well as the final new image. Should be cool right?

And yet, even though I have deterministically released resources (the whole point of dispose), I have a memory leak!

![memLeakImage.](http://onoffswitch.net/wp-content/uploads/2013/04/memLeakImage..png)

Weird...

What if I get rid of the fold and append everything in one go?

[csharp]  
public static Image Append(this Image source, List\<Image\> appends)  
{  
 var newImage = new Bitmap(source.Width + appends.Count() \* source.Width, source.Height);  
 using (var g = Graphics.FromImage(newImage))  
 {  
 g.DrawImage(source, 0, 0);

int i = 1;  
 foreach (var item in appends)  
 {  
 g.DrawImage(item, i \* source.Width, 0);

i++;  
 }  
 }

return newImage;  
}

private static void Main(string[] args)  
{  
 var src = @"C:\users\anton.kropp\desktop\bigImage.jpg";

var images = Enumerable.Repeat(Image.FromFile(src), 25).ToList();

var appendedImage = images.First().Append(images.Skip(1).ToList());

foreach (var image in images)  
 {  
 image.Dispose();  
 }

appendedImage.Dispose();

Console.ReadLine();  
}  
[/csharp]

And now when I wait on the read at the end

![noMemLeakImage.](http://onoffswitch.net/wp-content/uploads/2013/04/noMemLeakImage..png)

No memory leak!

Something with that aggregate function is causing a leak. Well, when in doubt, go to the source. Here is the decompiled source for the `Aggregate` overload that uses the first element in the sequence as the seed

[csharp highlight="14"]  
[\_\_DynamicallyInvokable]  
public static TSource Aggregate\<TSource\>(this IEnumerable\<TSource\> source, Func\<TSource, TSource, TSource\> func)  
{  
 if (source == null)  
 throw Error.ArgumentNull("source");  
 if (func == null)  
 throw Error.ArgumentNull("func");  
 using (IEnumerator\<TSource\> enumerator = source.GetEnumerator())  
 {  
 if (!enumerator.MoveNext())  
 throw Error.NoElements();  
 TSource source1 = enumerator.Current;  
 while (enumerator.MoveNext())  
 source1 = func(source1, enumerator.Current);  
 return source1;  
 }  
}  
[/csharp]

Once I saw this I spotted the leak right away. The issue is that each time an item is folded, the function is overwriting the reference to the first element with the aggregation lambda (highlighted). Since the `Image` class wraps native GDI+ code, this means that unless you call dispose on each instance that is created, the underlying unmanaged resources are still sitting around!

The reason the second version works is because I am not making intermediary images via an accumulator. Each image reference that is created is tracked then properly destroyed.

Now, understanding the issue, I was able to mimic the problem outside of the aggregate function:

[csharp]  
public static void MemLeak()  
{  
 var src = @"C:\users\anton.kropp\desktop\bigImage.jpg";

Image image1 = null;

foreach (var i in Enumerable.Range(0, 10))  
 {  
 image1 = Image.FromFile(src);  
 }

image1.Dispose();

Console.ReadLine();  
}  
[/csharp]

Just because it's the same variable, does not mean it's the same reference. Actually if we modify the code like this we can really drive the point home:

[csharp]  
public static void MemLeak()  
{  
 var src = @"C:\users\anton.kropp\desktop\bigImage.jpg";

Image image1 = null;

Image previousImage;

foreach (var i in Enumerable.Range(0, 10))  
 {  
 previousImage = image1;

image1 = Image.FromFile(src);

Console.WriteLine("References equal? " + ReferenceEquals(previousImage, image1));  
 }

image1.Dispose();

Console.ReadLine();  
}

[/csharp]

Which prints out

[code]  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
References equal? False  
[/code]

Which makes sense.

## Conclusion

What I got out of all of this is that you really need to know where your references are and understand what your temporary values are doing. Also, don't use disposable items as accumulators in an aggregator.

This is where I really regret posting the last article as a response to functional. If things were immutable by default I wouldn't have run into this problem. :)

