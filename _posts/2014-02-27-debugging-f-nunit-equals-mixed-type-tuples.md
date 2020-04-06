---
layout: post
title: Debugging F# NUnit equals for mixed type tuples
date: 2014-02-27 00:49:18.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- equality
- F#
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554308616;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3735;}i:1;a:1:{s:2:"id";i:4411;}i:2;a:1:{s:2:"id";i:2735;}}}}

permalink: "/2014/02/27/debugging-f-nunit-equals-mixed-type-tuples/"
---
Twitter user [Richard Dalton](https://twitter.com/richardadalton) asked a great question recently:

![Twitter question](http://onoffswitch.net/wp-content/uploads/2014/02/Screen-Shot-2014-02-26-at-4.08.38-PM.png)

And after a bit more digging he then mentioned

![Twitter question](http://onoffswitch.net/wp-content/uploads/2014/02/Screen-Shot-2014-02-26-at-4.10.17-PM.png)

Interesting. I downloaded the NUnit source and saw this:

[csharp highlight="61"]  
public bool AreEqual(object x, object y, ref Tolerance tolerance)  
{  
 this.failurePoints = new List\<FailurePoint\>();

if (x == null && y == null)  
 return true;

if (x == null || y == null)  
 return false;

if (object.ReferenceEquals(x, y))  
 return true;

Type xType = x.GetType();  
 Type yType = y.GetType();

EqualityAdapter externalComparer = GetExternalComparer(x, y);  
 if (externalComparer != null)  
 return externalComparer.AreEqual(x, y);

if (xType.IsArray && yType.IsArray && !compareAsCollection)  
 return ArraysEqual((Array)x, (Array)y, ref tolerance);

if (x is IDictionary && y is IDictionary)  
 return DictionariesEqual((IDictionary)x, (IDictionary)y, ref tolerance);

//if (x is ICollection && y is ICollection)  
 // return CollectionsEqual((ICollection)x, (ICollection)y, ref tolerance);

if (x is IEnumerable && y is IEnumerable && !(x is string && y is string))  
 return EnumerablesEqual((IEnumerable)x, (IEnumerable)y, ref tolerance);

if (x is string && y is string)  
 return StringsEqual((string)x, (string)y);

if (x is Stream && y is Stream)  
 return StreamsEqual((Stream)x, (Stream)y);

if (x is DirectoryInfo && y is DirectoryInfo)  
 return DirectoriesEqual((DirectoryInfo)x, (DirectoryInfo)y);

if (Numerics.IsNumericType(x) && Numerics.IsNumericType(y))  
 return Numerics.AreEqual(x, y, ref tolerance);

if (tolerance != null && tolerance.Value is TimeSpan)  
 {  
 TimeSpan amount = (TimeSpan)tolerance.Value;

if (x is DateTime && y is DateTime)  
 return ((DateTime)x - (DateTime)y).Duration() \<= amount;

if (x is TimeSpan && y is TimeSpan)  
 return ((TimeSpan)x - (TimeSpan)y).Duration() \<= amount;  
 }

if (FirstImplementsIEquatableOfSecond(xType, yType))  
 return InvokeFirstIEquatableEqualsSecond(x, y);  
 else if (xType != yType && FirstImplementsIEquatableOfSecond(yType, xType))  
 return InvokeFirstIEquatableEqualsSecond(y, x);

return x.Equals(y);  
}  
[/csharp]

That last line tipped me off. So lets look at some tests now:

[fsharp]  
\> z;;

val it : int [] \* int = ([|1|], 1)

\> p;;

val it : int [] \* int = ([|1|], 1)

\> z.Equals(p);;

val it : bool = false

\> z = p;;

val it : bool = true  
[/fsharp]

So that makes sense, there's no custom equality comparer for tuples, and since the references are different the obj equals fails. But why do the other things Richard said hold true then?

Well arrays have their own custom comparer that compares contents, that much is visible in the NUnit source. And tuples look to generate the same hash code IF they have value types in them, which you can test in fsi.

[fsharp]  
\> let ref1 = ([], 2);;

val ref1 : 'a list \* int

\> let ref2 = ([], 2);;

val ref2 : 'a list \* int

\> ref1.GetHashCode();;

val it : int = 2

\> ref2.GetHashCode();;

val it : int = 2  
[/fsharp]

But if a tuple contains a reference type

[fsharp]  
\> let ref3 = ([||], 1);;

val ref3 : 'a [] \* int

\> let ref4 = ([||], 1);;

val ref4 : 'a [] \* int

\> ref3.GetHashCode();;

val it : int = -1869554978

\> ref4.GetHashCode();;

val it : int = -259699334  
[/fsharp]

Suddenly they aren't equal! Arrays, being a built in .NET primitive, follow [these semantics](http://stackoverflow.com/questions/720177/default-implementation-for-object-gethashcode) for generating a hash code. Basically, they return a different value per each instance of the object in the app domain.

Now why does the f# `=` operator work? From the [source](https://github.com/fsharp/fsharp/blob/master/src/fsharp/FSharp.Core/prim-types.fs#L1975), it looks like they have created custom comparators for f# types which does structural equality:

[fsharp]  
// Note: because these FastEqualsTupleN functions are devirtualized by (=), they have PER semantics  
let inline FastEqualsTuple2 (comparer:System.Collections.IEqualityComparer) (x1,x2) (y1,y2) =  
 GenericEqualityWithComparerFast comparer x1 y1 &&  
 GenericEqualityWithComparerFast comparer x2 y2

let inline FastEqualsTuple3 (comparer:System.Collections.IEqualityComparer) (x1,x2,x3) (y1,y2,y3) =  
 GenericEqualityWithComparerFast comparer x1 y1 &&  
 GenericEqualityWithComparerFast comparer x2 y2 &&  
 GenericEqualityWithComparerFast comparer x3 y3

// .... etc ...

let inline (=) x y = GenericEquality x y  
[/fsharp]

Neat! I love it when things make sense.

