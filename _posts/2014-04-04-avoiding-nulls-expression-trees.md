---
layout: post
title: Avoiding nulls with expression trees
date: 2014-04-04 04:23:28.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- ast
- c#
- expression tree
- 'null'
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561472234;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:3779;}i:2;a:1:{s:2:"id";i:4862;}}}}

permalink: "/2014/04/04/avoiding-nulls-expression-trees/"
---
I've blogged about this subject [before](http://onoffswitch.net/minimizing-null-ref/), but I REALLY hate null refs. This is one of the reasons I love F# and other functional languages, null ref's almost never happen. But, in the real world I work as a C# dev and have to live with C#'s... nuisances.

In the other post, a big problem with the dynamic proxy was that it only worked with virtual methods, so it wasn't really all that practical. This time around I decided to try a different route and leverage expression tree's to actually build out the if checks automatically.

For the impatient, full source available at my [github](https://github.com/devshorts/MonadicNull)and the library is available on [nuget](https://www.nuget.org/packages/Devshorts.MonadicNull/0.2.1)

## Demonstration

Let me demonstrate the final usage first. If all of users properties and methods return null, executing this whole chain would fail starting at the null result of `GetSchool()`. But, by using the Option static class we can safely deconstruct and inspect the expression, returning if the chain is valid, what failed in it, and what was the final value (if one existed).

[csharp]  
public void TestGetSafe()  
{  
 var user = new User();

MethodValue\<string\> name = Option.Safe(() =\> user.GetSchool().District.Street.Name);

Assert.IsFalse(name.ValidChain());  
}  
[/csharp]

The lambda is converted to an expression tree and rebuilt into this glorious mess:

[code]  
.Lambda #Lambda1\<System.Func`2[NoNulls.Tests.SampleData.User,Devshorts.MonadicNull.MethodValue`1[System.String]]\>(NoNulls.Tests.SampleData.User $u)  
{  
 .Block() {  
 .Block(NoNulls.Tests.SampleData.User $var1) {  
 $var1 = $u;  
 .If ($var1 == null) {  
 .New Devshorts.MonadicNull.MethodValue`1[System.String](  
 null,  
 "u",  
 False)  
 } .Else {  
 .Block(NoNulls.Tests.SampleData.School $var2) {  
 $var2 = .Call $var1.GetSchool();  
 .If ($var2 == null) {  
 .New Devshorts.MonadicNull.MethodValue`1[System.String](  
 null,  
 "u.GetSchool()",  
 False)  
 } .Else {  
 .Block(NoNulls.Tests.SampleData.District $var3) {  
 $var3 = $var2.District;  
 .If ($var3 == null) {  
 .New Devshorts.MonadicNull.MethodValue`1[System.String](  
 null,  
 "u.GetSchool().District",  
 False)  
 } .Else {  
 .Block(NoNulls.Tests.SampleData.Street $var4) {  
 $var4 = $var3.Street;  
 .If ($var4 == null) {  
 .New Devshorts.MonadicNull.MethodValue`1[System.String](  
 null,  
 "u.GetSchool().District.Street",  
 False)  
 } .Else {  
 .Block(System.String $var5) {  
 $var5 = $var4.Name;  
 .New Devshorts.MonadicNull.MethodValue`1[System.String](  
 $var5,  
 "u.GetSchool().District.Street.Name",  
 True)  
 }  
 }  
 }  
 }  
 }  
 }  
 }  
 }  
 }  
 }  
}  
[/code]

The return value of the `Safe` method is a new object called `MethodValue` which tells you if the expression was successful (i.e. no nulls), or if it did fail, _what_ failed (i.e. where in the expression it failed). And of course if the chain is valid you can get the value safely.

## The magic

The magic is actually pretty easy. Because we have access to the expression tree we can rewrite it however we want. If you aren't familiar with expression trees, the idea is that the compiler can convert a lambda into it's abstract syntax tree. That means you can traverse it (with a visitor) and know information about the objects.

Since we need to know the information about the whole call chain we need to iterate over it all first. As we iterate we should track of what happened in the call chain so we can re-iterate over it later and manipulate it at the end.

[csharp]  
private readonly Stack\<Expression\> \_expressions = new Stack\<Expression\>();

private Expression \_finalExpression;

private void CaptureFinalExpression(Expression node)  
{  
 if (\_finalExpression == null)  
 {  
 \_finalExpression = node;  
 }  
}

protected override Expression VisitLambda\<Y\>(Expression\<Y\> node)  
{  
 base.Visit(node.Body);

CaptureFinalExpression(node.Body);

if (node.Parameters.Count \> 0)  
 {  
 \_expressions.Push(node.Parameters.First());

var final = BuildFinalStatement();

return Expression.Lambda(final, node.Parameters);  
 }

return Expression.Lambda(BuildFinalStatement());  
}

protected override Expression VisitMethodCall(MethodCallExpression node)  
{  
 \_expressions.Push(node);

return Visit(node.Object);  
}

protected override Expression VisitMember(MemberExpression node)  
{  
 \_expressions.Push(node);

return Visit(node.Expression);  
}  
[/csharp]

In the process of visiting the nodes, we pushed each piece of the call chain into a stack. Now we can build it all back out. The bulk of the work is in this recursive function. It iterates back through the stack, maps the previous call to a variable, and builds out if not null checks.

[csharp]  
private Expression BuildIfs(Expression current, Expression prev = null)  
{  
 var stringRepresentation = Expression.Constant(current.ToString(), typeof(string));

var variable = Expression.Parameter(current.Type, NextVarName);

Expression evaluatedExpression = EvaluateExpression(current, prev);

var assignment = Expression.Assign(variable, evaluatedExpression);

var end = \_expressions.Count == 0;

var nextExpression =  
 !end  
 ? BuildIfs(\_expressions.Pop(), variable)  
 : LastExpression(variable, stringRepresentation);

Expression blockBody;

if (!end)  
 {  
 var whenNull = OnNull(stringRepresentation);

blockBody = CheckForNull(variable, whenNull, nextExpression);  
 }  
 else  
 {  
 blockBody = nextExpression;  
 }

return Expression.Block(new [] { variable }, new[] { assignment, blockBody });  
}  
[/csharp]

When I say "map the previous call" I mean building out something like this:

[csharp]  
var var1 = user;  
if(var1 != null){  
 var var2 = var1.school

if(var2 != null)  
 ....  
}  
[/csharp]

The initial statement "user" needs to get assigned to a variable. Then this variable needs to be transformed into a new call where we call ".school" on it. We know what to do with ".school" because the call for ".school" was captured as part of the stack iteration. During the iteration of the expression tree in the visitor we were able to capture each portion of the tree.

Look:

![2014-04-03 21_14_26-](http://onoffswitch.net/wp-content/uploads/2014/04/2014-04-03-21_14_26-.png)

Given that we have each piece we can now inspect it and manipulate other pieces with it

![2014-04-03 20_58_12-NoNulls (Debugging) - Microsoft Visual Studio (Administrator)](http://onoffswitch.net/wp-content/uploads/2014/04/2014-04-03-20_58_12-NoNulls-Debugging-Microsoft-Visual-Studio-Administrator.png)

Now lets assign it to a variable

![2014-04-03 20_59_07-NoNulls (Debugging) - Microsoft Visual Studio (Administrator)](http://onoffswitch.net/wp-content/uploads/2014/04/2014-04-03-20_59_07-NoNulls-Debugging-Microsoft-Visual-Studio-Administrator.png)

From here on out we've cached the value.

The evaluate expression function is pretty simple:

[csharp]  
private Expression EvaluateExpression(Expression current, Expression prev)  
{  
 if (prev == null)  
 {  
 return current;  
 }

if (current is MethodCallExpression)  
 {  
 var method = current as MethodCallExpression;

return Expression.Call(prev, method.Method, method.Arguments);  
 }

if (current is MemberExpression)  
 {  
 var member = current as MemberExpression;

return Expression.MakeMemberAccess(prev, member.Member);  
 }

return current;  
}  
[/csharp]

The rest of the work is boilerplate of creating if checks and returning the final result when necessary

[csharp]  
private Expression OnNull(ConstantExpression stringRepresentation)  
{  
 var falseVal = Expression.Constant(false);

var nullValue = Expression.Constant(default(T), \_finalExpression.Type);

return Expression.New(MethodValueConstructor, new Expression[] { nullValue, stringRepresentation, falseVal });  
}

private Expression CheckForNull(ParameterExpression variable, Expression whenNull, Expression nextExpression)  
{  
 var ifNull = Expression.ReferenceEqual(variable, Expression.Constant(null));

return Expression.Condition(ifNull, whenNull, nextExpression);  
}

private Expression LastExpression(ParameterExpression variable, ConstantExpression stringRepresentation)  
{  
 var trueVal = Expression.Constant(true);

return Expression.New(MethodValueConstructor, new Expression[] { variable, stringRepresentation, trueVal });  
}  
[/csharp]

## Conclusion

Building expression tree's was a little complicated. There aren't any return statements, and I ran into a lot of weird errors assigning and accessing variables (you need to use certain overloads of the block expression, which I only figured out after reading a [jon skeet answer](http://stackoverflow.com/a/3370894/310196) on stack overflow). Still, the resulting code is concise and clean, and until the .? operator shows up this isn't a bad alternative!

Full source is available at my [github](https://github.com/devshorts/MonadicNull)

