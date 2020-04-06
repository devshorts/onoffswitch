---
layout: post
title: Creating stronger value type contracts
date: 2015-01-05 19:35:56.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- dapper
- types
- wcf
- webapi
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561600152;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:4411;}i:2;a:1:{s:2:"id";i:3899;}}}}

permalink: "/2015/01/05/creating-stronger-type-contracts/"
---
I've long been annoyed that value types don't have strong semantic information attached to them such that the compiler would barf if I try and pass an value type that isn't semantically the same as what the function wanted. For example, what does the following signature mean other than than taking in 2 ints and returning a bool?

[code]  
IsLoggedIn :: int -\> int -\> bool  
[/code]

What I'd really like the signature to look like is

[code]  
IsLoggedIn :: UserId -\> SessionId -\> bool  
[/code]

In F# you can do this sort of with type aliases and augmenting the signature with the type information. However, its just editor magic, it doesn't actually compile to anything that would stop you from accidentally calling a function with the arguments reversed. An int is an int is an int, right?

[code]  
var userId = 1  
var sessionId = 2

IsLoggedIn(sessionId, userId)  
[/code]

This is perfectly valid to the compiler and means you won't catch it until its unit tested, peer reviewed (if everyone is paying attention), or found during runtime. I'd like to have this caught at compile time.

## A solution

Instead, what if we wrapped important value types into their own structs? Something like this:

[csharp]  
public struct UserId  
{  
 private readonly Int32 \_int;

private UserId(int @int)  
 {  
 \_int = @int;  
 }

public static explicit operator UserId(int value)  
 {  
 return new UserId(value);  
 }

public static implicit operator int(UserId value)  
 {  
 return value.\_int;  
 }  
}

public struct SessionId  
{  
 private readonly Int32 \_int;

private SessionId(int @int)  
 {  
 \_int = @int;  
 }

public static explicit operator SessionId(int value)  
 {  
 return new SessionId(value);  
 }

public static implicit operator int(SessionId value)  
 {  
 return value.\_int;  
 }  
}  
[/csharp]

Structurally its exactly the same as an int, and it doesn't cost you anything to use it. But, now the compiler will fail if you try and pass this

![strongtypes](http://onoffswitch.net/wp-content/uploads/2015/01/strongtypes.png)

## Adding some meta data

Now there is a problem though of project boundaries. What I mean is when your data types go over the wire, either through webapi or wcf or to a DB or some other form. To make this actually useful we have to introduce a few more things to make it easy to be transparent through these edge cases.

First, lets augment the strong inheritance heirarchy. I have interfaces that look like:

[csharp]  
public interface IStrongType\<out T\> : IAcceptStrongTypeVisitor  
{  
 T UnderlyingValue();  
}  
[/csharp]

[csharp]  
public interface IInt32 : IStrongType\<int\>  
{  
 string ToString(string format);

string ToString(string format, IFormatProvider provider);

string ToString(IFormatProvider provider);  
}  
[/csharp]

[csharp]  
public interface IStrongTypeVistor\<out TResult\>  
{  
 TResult Visit(IGuid data);  
 TResult Visit(IInt32 data);  
 TResult Visit(IInt64 data);  
 TResult Visit(IFloat data);  
 TResult Visit(IDouble data);  
}  
[/csharp]

Now I can have strong types implement their corresponding value type interfaces (I want a strong int to have the same methods as a regular int) as well as visit on top of them.

## Serializing JSON

The first order of business is serializing to and from JSON. To do that I've created a json converter that knows how to cast and get the underlying value of a strong type

[csharp]  
public class PrimitiveJsonConverter\<T\> : JsonConverter  
 where T : struct  
{  
 public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)  
 {  
 var underlying = (IStrongType\<T\>)(value);

writer.WriteValue(underlying.UnderlyingValue());  
 }

public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)  
 {  
 var data = serializer.Deserialize\<T\>(reader);

return data.ExplicitCastTo(objectType);  
 }

public override bool CanConvert(Type objectType)  
 {  
 return typeof(T).IsAssignableFrom(objectType);  
 }  
}  
[/csharp]

And a utility extension method to reflectively invoke the explicit cast operator on a type  
[csharp]  
public static object ExplicitCastTo(this object obj, Type type)  
{  
 var castOperator = type.GetMethod("op\_Explicit", new[] { obj.GetType() });

if (castOperator == null)  
 {  
 throw new InvalidCastException("Can't cast to " + type.Name);  
 }

return castOperator.Invoke(null, new[] { obj });  
}  
[/csharp]

Now a strong type will actually look like this:

[csharp]  
[Serializable]  
[JsonConverter(typeof(IntCaster))]  
public struct SessionId : IInt32  
{  
 private readonly Int32 \_int;

private SessionId(int @int)  
 {  
 \_int = @int;  
 }

public static explicit operator SessionId(int value)  
 {  
 return new SessionId(value);  
 }

public static implicit operator int(SessionId value)  
 {  
 return value.\_int;  
 }

public int UnderlyingValue()  
 {  
 return \_int;  
 }

public TResult Accept\<TResult\>(IStrongTypeVistor\<TResult\> visitor)  
 {  
 return visitor.Visit(this);  
 }

public override string ToString()  
 {  
 return \_int.ToString();  
 }

public string ToString(string format)  
 {  
 return \_int.ToString(format);  
 }

public string ToString(string format, IFormatProvider provider)  
 {  
 return \_int.ToString(format, provider);  
 }

public string ToString(IFormatProvider provider)  
 {  
 return \_int.ToString(provider);  
 }  
}  
[/csharp]

## Handling WCF

For WCF I took a lazier approach. For my use case we can distribute an assembly that contains all the strong types and ask consumers to re-use the types when generating proxies. If they don't, WCF will auto generate a type that boxes the result and send it to you anyways. It makes it more annoying for a consumer, but not impossible to use.

## WebAPI parameter parsing

Web Api exposes a way to hook into the parameter/object binding as things come in over the wire. You just need to implement the `HttpParameterBinding` abstract class. Then you register the binder either as an attribute on your data object, or as an attribute on the parameter in the web api method, or via registration at startup.

First I'll show the parameter binder base class.

[csharp]  
public class PrimitiveBinder\<T\> : HttpParameterBinding where T : struct  
{  
 public PrimitiveBinder(HttpParameterDescriptor descriptor)  
 : base(descriptor)  
 {  
 }

public override Task ExecuteBindingAsync(ModelMetadataProvider metadataProvider, HttpActionContext actionContext, CancellationToken cancellationToken)  
 {  
 var value = actionContext.RequestContext.RouteData.Values[Descriptor.ParameterName];

var @struct = TypeDescriptor.GetConverter(typeof(T)).ConvertFromString(value.ToString());

actionContext.ActionArguments[Descriptor.ParameterName] = @struct.ExplicitCastTo(Descriptor.ParameterType);

var tsc = new TaskCompletionSource\<object\>();  
 tsc.SetResult(null);  
 return tsc.Task;  
 }  
}  
[/csharp]

You can see that it converts the string representation into the raw value type (which is T), then it casts the raw value type into the type of the parameter

Each strong type wrapper will have its own implementation of the binder where it passes in the underlying value type of T. As an example, here is int:

[csharp]  
public class IntBinder : PrimitiveBinder\<int\>  
{  
 public IntBinder(HttpParameterDescriptor descriptor)  
 : base(descriptor)  
 {  
 }  
}  
[/csharp]

When the application boots up it calls the registration on the http config

[csharp]  
public static void RegisterStrongTypes(HttpConfiguration config)  
{  
 config.ParameterBindingRules.Add(FindDescriptor);  
}

private static HttpParameterBinding FindDescriptor(HttpParameterDescriptor descriptor)  
{  
 if (typeof(IGuid).IsAssignableFrom(descriptor.ParameterType))  
 {  
 return new GuidBinder(descriptor);  
 }

if (typeof(IInt32).IsAssignableFrom(descriptor.ParameterType))  
 {  
 return new IntBinder(descriptor);  
 }

if (typeof(IFloat).IsAssignableFrom(descriptor.ParameterType))  
 {  
 return new FloatBinder(descriptor);  
 }

return null;  
}  
[/csharp]

Now web api knows what to do based on the tagged interface of the strong type (in the int example, they are all of IInt32 interfaces).

As an example, if this is our method on the controller

[csharp]  
[Route("int/{test}"), HttpGet]  
public IntExample Test(IntExample test)  
{  
 return test;  
}  
[/csharp]

We can pass in an int

[code]  
localhost/api/int/1  
[/code]

And get the same value echoed back out to us without having to add any annotations to either the controller, the method, or the parameters.

## Dapper

For the project I work on, we also use dapper as our micro ORM. This is great since it lets us pass in anonymous objects representing stored procedure parameters and it can auto map into objects and primitives for us. However, dapper has no idea what to do with our strong types so we have to augment its type serializer.

This is where the visitor interface comes into play. On application load, we can leverage a registration function that tells dapper what to do for the specific types. The one thorn here is that dapper can't map an interface type to a type handler, it needs to be for each type explicity. Thats OK since we can reflectively find all types that implement the root IStrongType<t> interface and register them automatically</t>

[csharp]  
internal class StrongTypeMapper : SqlMapper.ITypeHandler  
{  
 public void SetValue(IDbDataParameter parameter, object value)  
 {  
 parameter.DbType = new DbTypeStrongVisitor().Visit(value as IAcceptStrongTypeVisitor);

parameter.Value = new UnderlyingStrongTypeVisitor().Visit(value as IAcceptStrongTypeVisitor);  
 }

public object Parse(Type destinationType, object value)  
 {  
 return value.ExplicitCastTo(destinationType);  
 }  
}

internal class DbTypeStrongVisitor : IStrongTypeVistor\<DbType\>  
{

public DbType Visit(IGuid guid)  
 {  
 return DbType.Guid;  
 }

public DbType Visit(IInt32 data)  
 {  
 return DbType.Int32;  
 }

public DbType Visit(IInt64 data)  
 {  
 return DbType.Int64;  
 }

public DbType Visit(IFloat data)  
 {  
 return DbType.Double;  
 }

public DbType Visit(IDouble data)  
 {  
 return DbType.Decimal;  
 }

public DbType Visit(IAcceptStrongTypeVisitor visitor)  
 {  
 return visitor.Accept(this);  
 }  
}  
[/csharp]

And the initializer call:

[csharp]  
public static void InitTypeMappings()  
{  
 Assembly.GetExecutingAssembly()  
 .FindStrongTypes()  
 .ForEach(i =\> SqlMapper.AddTypeHandler(i, new StrongTypeMapper()));  
}  
[/csharp]

Where `FindStrongTypes` finds all the types that implement the generic interface of IStrongType<t>.</t>

## But the boilerplate!

This is great and all, but kind of annoying to manage by hand. It's a lot of boilerplate to write for what used to just be an int or float or guid. To combat this, my team is using a custom code generator that auto generates strong types along with the visitor information, and a bunch of other auto-gend data for our codebase. It'd be easy to write your own, since the template is pretty much exactly the same.

