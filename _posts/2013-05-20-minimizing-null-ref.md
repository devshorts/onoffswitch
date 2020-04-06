---
layout: post
title: Minimizing the null ref with dynamic proxies
date: 2013-05-20 08:00:33.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dynamic proxy
- maybe monad
- 'null'
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561365978;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4493;}i:1;a:1:{s:2:"id";i:3565;}i:2;a:1:{s:2:"id";i:3723;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/20/minimizing-null-ref/"
---
In a production application you frequently can find yourself working with objects that have a large accessor chain like

[code]  
student.School.District.Street.Name  
[/code]

But when you want to program defensively you need to always do null checks on any reference type. So your accessing chain looks more like this instead

[csharp]  
if (student.School != null)  
{  
 if (student.School.District != null)  
 {  
 if (student.School.District.Street != null)  
 {  
 s += student.School.District.Street.Name;  
 }  
 }  
}  
[/csharp]

Which sucks. Especially since its easy to forget to add a null check, and not to mention it clutters the code up. Even if you used an option type, you still have to check if it's something or if its nothing, and dealing with huge option chains is just as annoying.

One solution is to use the [maybe monad](http://devtalk.net/csharp/chained-null-checks-and-the-maybe-monad/), which can be implemented using extension methods and lambdas. While this is certainly better, it can still can get unwieldy.

What I really want is a way to just access the chain, and if any part of it is null for it to return null.

## The magic of Castle Dynamic Proxy

This is where the magic of [castle dynamic proxy](http://www.castleproject.org/projects/dynamicproxy/) comes into play. Castle creates [runtime byte code](http://www.codeproject.com/Articles/121568/Dynamic-Type-Using-Reflection-Emit#heading0002) that can subclass your class and intercept method calls to it. This means you can now control what happens each time a method is invoked on your function, both by manipulating the return value and by choosing whether or not to even invoke the function. Lots of libraries use castle to do neat things, like the [moq library](https://code.google.com/p/moq/) from google and [NHibernate](http://nhforge.org/).

For my purposes, I wanted to create a null safe proxy that lets me safely iterate through the function call chain. Before I dive into it, lets see what the final result is:

[csharp]  
var user = new User();

var name = user.NeverNull().School.District.Street.Name.Final();  
[/csharp]

At this point `name` can be either null, or the street name. But since this user never set any of its public properties everything is null, so name here will be null. At this point I can do one null check and move on.

## The start

`NeverNull` is an extension method that wraps the invocation target (the thing calling the method) with a new dynamic proxy.

[csharp]  
public static T NeverNull\<T\>(this T source) where T : class  
{  
 return (T) \_generator.CreateClassProxyWithTarget(typeof(T), new[] { typeof(IUnBoxProxy) }, source, new NeverNullInterceptor(source));  
}  
[/csharp]

I'm doing a few things here. First I'm making a proxy that wraps the source object. The proxy will be of the same type as the source. Second, I'm telling castle to also add the `IUnBoxProxy` interface to the proxy implementation. We'll see why that's used later. All it means is that the proxy that is returned implements not only all the methods of the source, but is also going to be of the `IUnBoxProxy` interface. Third, I am telling castle to use a `NeverNullInterceptor` that holds a reference to the source item. This interceptor is responsible for manipulating any function calls on the source object.

## The method interceptor

The interceptor isn't that complicated. Here is the whole class:

[csharp]  
public class NeverNullInterceptor : IInterceptor  
{  
 private object Source { get; set; }

public NeverNullInterceptor(object source)  
 {  
 Source = source;  
 }

public void Intercept(IInvocation invocation)  
 {  
 try  
 {  
 if (invocation.Method.DeclaringType == typeof(IUnBoxProxy))  
 {  
 invocation.ReturnValue = Source;  
 return;  
 }

invocation.Proceed();

var returnItem = Convert.ChangeType(invocation.ReturnValue, invocation.Method.ReturnType);

if (!PrimitiveTypes.Test(invocation.Method.ReturnType))  
 {  
 invocation.ReturnValue = invocation.ReturnValue == null  
 ? ProxyExtensions.NeverNullProxy(invocation.Method.ReturnType)  
 : ProxyExtensions.NeverNull(returnItem, invocation.Method.ReturnType);  
 }  
 }  
 catch (Exception ex)  
 {  
 invocation.ReturnValue = null;  
 }  
 }  
}  
[/csharp]

The main gist of this class is that whenever a function gets called on a proxy object, the interceptor can capture the function call. We created the specific proxy to be tied to this interceptor as part of the proxy generation.

When a function is captured by the interceptor, the interceptor can choose to invoke the actual underlying function if it wants to (via the proceed method). After that, the interceptor tests to see if the function return value was null or not. If the value wasn't null, the interceptor then proxies the return value (creating a chain of proxy objects). This means that the next function call in the accessor chain is now also on a proxy!

But, if the return value was null we still need to continue the accessor chain. Unlike the maybe monad, we can't bail in the middle of the call. So, what we do instead is to create an empty proxy of the same type. This just gives us a way to capture invocations onto what would otherwise be a null object. Castle can give you a proxy that doesn't wrap any target. This is what moq does as well. If anyone calls a function on this proxy, the interceptor's intercept method gets called and we can choose to not proceed with the actual invocation! There's no underlying wrapped target, it's just the interceptor catching calls.

In the scenario where the return result is null, here is the function to proxy the type

[csharp]  
public static object NeverNullProxy(Type t)  
{  
 return \_generator.CreateClassProxy(t, new[] { typeof(IUnBoxProxy) }, new NeverNullInterceptor(null));  
}  
[/csharp]

Now, you may notice that I'm passing `null` to the constructor of the interceptor, but previously I passed a source object to the constructor. This is because I want the interceptor to know what is the underlying proxied target. This is how I'm going to be able to unbox the final value out of the proxy chain when it's requested. This is also the reason for the `IUnBoxProxy` interface we added.

## Getting the value out!

At this point there is an entire proxy chain set up. Once you enter the proxy chain, all other functions on that object are also proxies. But at some point you want to get the actual value out, whether its null or not. This is where that special interface comes in. Using an extension method on all object types we can cast the object to the special interface (remembering that the object we're working on is actually a proxy and that it should have implemented the special interface we told it to) and execute a function on it. It really doesn't matter which function, just a function

[csharp]  
public static T Final\<T\>(this T source)  
{  
 var proxy = (source as IUnBoxProxy);  
 if (proxy == null)  
 {  
 return source;  
 }

return (T)proxy.Value;  
}  
[/csharp]

Since the proxy is actually a dynamic proxy that was created we get caught back in the interceptor. This is why this block exists

[csharp]  
if (invocation.Method.DeclaringType == typeof(IUnBoxProxy))  
{  
 invocation.ReturnValue = Source;  
 return;  
}  
[/csharp]

If the declaring type (i.e. the thing calling the function) is of that type (which it is since we explicitly cast it to it) then return the internal stored unboxed proxy. If the proxy contained null then a null gets returned, otherwise the last thing in the chain gets returned.

I specificailly excluded primitives during the proxy boxing phase since a primitive implies the final ending of the chain. That and castle kept throwing me an error saying that it

[csharp]  
Could not load type 'Castle.Proxies.StringProxy' from assembly 'DynamicProxyGenAssembly2, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null' because the parent type is sealed.  
[/csharp]

But thats OK since we don't need to proxy primitives in this scenario.

## Performance tests

Now this is great and all, but if it incurs an enormous performance penalty then we can't really use it. This is where I ran some unscientific tests. In a unit test run in release I checked the relative execution time of the following 3 functions:

Create an empty user and use the never null proxy to check a string some amount of times. The console writeline exists only to make sure the compiler doesn't optimize out unused variables.

[csharp]  
private void NullWithProxy(int amount)  
{  
 var user = new User();

var s = "na";  
 for (int i = 0; i \< amount; i++)  
 {  
 s += user.NeverNull().School.District.Street.Name.Final() ?? "na";  
 }

Console.WriteLine(s.FirstOrDefault());  
}  
[/csharp]

Test a non null object chain with the proxy

[csharp]  
 private void TestNonNullWithProxy(int amount)  
 {  
 var student = new User  
 {  
 School = new School  
 {  
 District = new District  
 {  
 Street = new Street  
 {  
 Name = "Elm"  
 }  
 }  
 }  
 };

var s = "na";  
 for (int i = 0; i \< amount; i++)  
 {  
 s += student.NeverNull().School.District.Street.Name.Final();  
 }

Console.WriteLine(s.FirstOrDefault());  
 }  
[/csharp]

And finally test a bunch of if statements on a non null object

[csharp]  
private void NonNullNoProxy(int amount)  
{  
 var student = new User  
 {  
 School = new School  
 {  
 District = new District  
 {  
 Street = new Street  
 {  
 Name = "Elm"  
 }  
 }  
 }  
 };

var s = "na";  
 for (int i = 0; i \< amount; i++)  
 {  
 if (student.School != null)  
 {  
 if (student.School.District != null)  
 {  
 if (student.School.District.Street != null)  
 {  
 s += student.School.District.Street.Name;  
 }  
 }  
 }  
 }

Console.WriteLine(s.FirstOrDefault());  
}  
[/csharp]

And the results are

[![performanceChart](http://onoffswitch.net/wp-content/uploads/2013/05/performanceChart-1024x692.png)](http://onoffswitch.net/wp-content/uploads/2013/05/performanceChart.png)

You can see on iteration 1 that there is a big spike in using the proxy. That's because castle has to initially create and then cache dynamic proxies. After that things level out and grow linearly. While you do incur a penalty hit, its not that far off from regular if checks. Doing 4 chained proxy checks 5000 times runs about 200 milliseconds, compared to 25 milliseconds with direct if checks. While its 8 times longer, you get the security of knowing you won't accidentally have a null reference exception. For lower amounts of accessing the time is pretty comparable.

## Conclusion

Unfortunately a downside to all of this is that castle can only proxy methods and properties that are marked as `virtual`. Also I had a lot of difficulty getting proxying of enumerables to work. I was only able to get it to work with things that are declared as `IEnumerable` or `List` but not Dictionary or HashSet or anything else. If you know how to do this please [let me know](http://stackoverflow.com/questions/16525589/interceptor-for-ienumerable-methods)! Because of those limitations I wouldn't suggest using this in a production application. But, maybe, one of these days a language will come out with this built in and I'll be pretty stoked about that.

For full source check out [my github](https://github.com/devshorts/Playground/tree/master/NoNulls). Also I'd like to thank my coworker [Faisal](http://fmansoor.wordpress.com/) for really helping out on this idea. It was his experience with dynamic proxies that led to this post.

