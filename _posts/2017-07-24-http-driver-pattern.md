---
layout: post
title: The HTTP driver pattern
date: 2017-07-24 23:30:07.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- http
- library
- patterns
- scala
meta:
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1557689075;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4939;}i:1;a:1:{s:2:"id";i:4945;}i:2;a:1:{s:2:"id";i:289;}}}}
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2017/07/24/http-driver-pattern/"
---
Yet another SOA blog post, this time about calling services. I've seen a lot of posts, articles, even books, on how to _write_ services but not a good way about _calling_ services. It may seem trivial, isn't calling a service a matter of making a web request to one? Yes, it is, but in a larger organization it's not always so trivial.

# Distributing fat clients

The problem I ran into was the service stack in use at my organization provided a feature rich client as an artifact of a services build. It had retries, metrics, tracing with zipkin, etc. But, it also pulled in things like finagle, netty, jackson, and each service may be distributing slightly different versions of all of these dependencies. When you start to consume 3, 4, 5 or more clients in your own service, suddenly you've gotten into an intractable mess of dependencies. Sometimes there's no actual way to resolve them all without forcing upgrades in other services! That... sucks. It violates the idea of services in that my service is now coupled to your service.

You don't want to force service owners to have to write clients for each service they want to call. That'd be a big waste of time and duplicated effort. If your organization is mono-lingual (i.e. all java/scala/whatever) then its still worth providing a feature rich client that has the sane things built in: retries, metrics, tracing, fast fail, serialization, etc. But you don't want services leaking all the nuts and bolts to each other.

One solution is to auto generate clients server side. This is akin to what [WCF](https://docs.microsoft.com/en-us/dotnet/framework/wcf/whats-wcf) does, or projects like [swagger](https://swagger.io/), [thrift](https://thrift.apache.org/) for RPC, etc. The downside here is that the generated code is usually pretty nasty and sometimes its hard to plug in to augment the clients with custom tracing, correlation tracking, etc. Other times the API itself might need a few nicety helper methods that you don't want to expose in the raw API itself. But in the auto generated world, you can't do this.

There are other projects like [Retrofit](http://square.github.io/retrofit/) that look like they solve the problem since your client is just an interface and its only dependency is [OkHttp](http://square.github.io/okhttp/). But retrofit isn't scala friendly (None's need custom support, default arguments in methods are not properly intercepted, etc). You're also bound to the back-compat story of retrofit/okhttp, assuming that they can do things like make sure older versions live side by side together.

In practice, I found that retrofit (even with scala's issues) didn't work well in a distributed services environment where everyone was at wildly different versions of things.

# Abstracting HTTP

However, taking the idea from retrofit we can abstract away http calls with an http driver. Http really isn't that complicated, especially for how its used in conjuction with service to service calls:

[scala]  
import scala.concurrent.{ExecutionContext, Future}

case class ApiRequest(  
 path: String,  
 queryParams: Seq[(String, Option[String])] = Nil,  
 headers: Seq[(String, Option[String])] = Nil,  
 options: Option[RequestOptions] = None  
)

case class RequestOptions(  
 contentType: Option[String],  
 characterSet: String = "utf-8"  
)

/\*\*  
 \* A response with a body  
 \*  
 \* @param data The deserialized data  
 \* @param response The raw http response  
 \* @tparam T The type to deserialize  
 \*/  
case class BodyResponse[T](data: T, response: RawResponse)

/\*\*  
 \* A raw response that contains code, the body and headers  
 \*  
 \* @param code  
 \* @param body  
 \* @param headers  
 \*/  
case class RawResponse(code: Int, body: String, headers: Map[String, List[String]])

/\*\*  
 \* An http error that all drivers should throw on non 2xx  
 \*  
 \* @param code The code  
 \* @param body An optional body  
 \* @param error The inner exception (may be driver specific)  
 \*/  
case class HttpError(code: Int, body: Option[String], error: Exception)  
 extends Exception(s"Error ${code}, body: ${body}", error)

/\*\*  
 \* Marker trait indicating an http client  
 \*/  
trait HttpClient

/\*\*  
 \* The simplest HTTP Driver. This is used to abstract libraries that call out over the wire.  
 \*  
 \* Anyone can create a driver as long as it implements this interface  
 \*/  
trait HttpDriver {  
 val serializer: HttpSerializer

def get[TRes: Manifest](  
 request: ApiRequest  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def post[TReq: Manifest, TRes: Manifest](  
 request: ApiRequest,  
 body: Option[TReq]  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def put[TReq: Manifest, TRes: Manifest](  
 request: ApiRequest,  
 body: Option[TReq]  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def patch[TReq: Manifest, TRes: Manifest](  
 request: ApiRequest,  
 body: Option[TReq]  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def custom[TReq: Manifest, TRes: Manifest](  
 method: Methods,  
 request: ApiRequest,  
 body: Option[TReq]  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def delete[TRes: Manifest](  
 request: ApiRequest  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]

def bytesRaw[TRes: Manifest](  
 method: Methods,  
 request: ApiRequest,  
 body: Option[Array[Byte]]  
 )(implicit executionContext: ExecutionContext): Future[BodyResponse[TRes]]  
}  
[/scala]

Service owners who want to distribute a client can create clients that have no dependencies (other than the driver definition. Platform maintainers, like myself, can be dilligent about making sure the driver interface _never breaks_, or if it does is broken in a new namespace such that different versions can peacefully co-exist in the same process.

An example client can now look like

[scala]  
class ServiceClient(driver: HttpDriver) {  
 def ping()(implicit executionContext: ExecutionContext): Future[Unit] = {  
 driver.get[Unit]("/health").map(\_.data)  
 }  
}  
[/scala]

But we still need to provide an implementation of a driver. This is where we can decouple things and provide drivers that are properly tooled with all the fatness we want (netty/finagle/zipkin tracing/monitoring/etc) and service owners can bind their clients to whatever driver they want. Those provided implementations can be in their own shared library that only service's bind to (not service clients! i.e. terminal endpoints in the dependency graph)

There are few advantages here:

- Clients can be distributed at multiple scala versions without dependency conflicts
- It's much simpler to version manage and back-compat an interface/trait than it is an entire lib
- Default drivers that _do the right thing_ can be provided by the service framework, and back compat doesn't need to be taken into account there since the only consumer is the service (it never leaks). 
- Drivers are simple to use, so if someone needs to roll their own client its really simple to do it

# Custom errors

We can do some other cool stuff now too, given we've abstracted away how to call http code. Another common issue with clients is dealing with meaningful errors that aren't just the basic http 5xx/4xx codes. For example, if you throw a 409 conflict you may want the client to actually receive a `WidgetInIncorrectState` exception for some calls, and in other calls maybe a `FooBarInUse` error that contains more semantic information. Basically overloading what a 409 means for a particular call/query. One way of doing this is with a discriminator in the error body:

[code lang=text]  
HTTP 409 response:  
{  
 "code": "WidgetInIncorrectState",  
 "widgetName: "foo",  
 "widgetSize": 1234  
}  
[/code]

Given we don't want client code pulling in a json library to do json parsing, the driver needs to support [context aware deserialization](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization).

To do that, I've exposed a `MultiType` object that defines

- Given a path into the json object, which field defines the discriminator
- Given a discriminator, which type to deserialize to
- Which http error code to apply all this too

And it looks like:

[code lang=scala]  
/\*\*  
 \* A type representing deserialization of multiple types.  
 \*  
 \* @param discriminatorField The field that represents the textual "key" of what the subtype is. Nested fields can be located using  
 \* json path format of / delimited. I.e /foo/bar  
 \* @param pathTypes The lookup of the result of the discriminatorField to the subtype mapper  
 \* @tparam T The supertype of all the subtypes  
 \*/  
case class MultiType[T](  
 discriminatorField: String,  
 pathTypes: Map[String, SubType[\_ \<: T]]  
)

/\*\*  
 \* Represents a subtype as part of a multitype mapping  
 \*  
 \* @param path The optional json sub path (slash delimited) to deserialize the type as.  
 \* @tparam T The type to deserialize  
 \*/  
case class SubType[T: Manifest](path: Option[String] = None) {  
 val clazz = manifest[T].runtimeClass.asInstanceOf[Class[T]]  
}  
[/code]

Using this in a client looks like:

[code lang=scala]  
class ServiceClient(driver: HttpDriver) {  
 val errorMappers = MultiType[ApiException](discriminatorField = "code", Map(  
 "invalidData" -\> SubType[InvalidDataException]()  
 ))

def ping()(implicit executionContext: ExecutionContext): Future[Unit] = {  
 driver.get[Unit]("/health").map(\_.data).failWithOnCode(500, errorMappers)  
 }  
}  
[/code]

This is saying that when I get the value `invalidData` in the json response of field `code` on an http 500 error, to actually throw an `InvalidDataException` in the client.

How does this work? Well just like the http driver, we've abstracted the serializer and that's all plugged in by the service consumer

[code lang=scala]  
case class DiscriminatorDoesntExistException(msg: String) extends Exception(msg)

object JacksonHttpSerializer {  
 implicit def jacksonToHttpSerializer(jacksonSerializer: JacksonSerializer): HttpSerializer = {  
 new JacksonHttpSerializer(jacksonSerializer)  
 }  
}

class JacksonHttpSerializer(jackson: JacksonSerializer = new JacksonSerializer()) extends HttpSerializer {  
 override def fromDiscriminator[SuperType](multiType: MultiType[SuperType])(str: String): SuperType = {  
 val tree = jackson.objectMapper.readTree(str)

val node = tree.at(addPrefix(multiType.discriminatorField, "/"))

val subType = multiType.pathTypes.get(node.textValue()).orElse(multiType.defaultType).getOrElse {  
 throw new RuntimeException(s"Discriminator ${multiType.discriminatorField} does not exist")  
 }

val treeToDeserialize = subType.path.map(m =\> tree.at(addPrefix(m, "/"))).getOrElse(tree)

jackson.objectMapper.treeToValue(treeToDeserialize, subType.clazz)  
 }

override def toString[T](data: T): String = {  
 jackson.toJson(data)  
 }

override def fromString[T: Manifest](str: String): T = {  
 jackson.fromJson(str)  
 }

private def addPrefix(s: String, p: String) = {  
 p + s.stripPrefix(p)  
 }  
}  
[/code]

# Inherent issues

While there are a lot of goodies in abstracting serialization and http calling into a library API provided with implementations (drivers), it does handicap the clients a little bit. Things like doing custom manipulation of the raw response, any sort of business logic, adding other libraries, etc is really frowned upon. I'd argue this is a good thing and that this should all be handled at the service level since a client is always a _nice to have_ and not a _requirement_.

# Conclusion

The ultimate goal in SOA is separation. But 100% separation should not mean copy-pasting things, reinventing the wheel, or not sharing any code. It just means you need to build the proper lightweight abstractions to help keep strong barriers between services without creating a distributed monolith.

With the http drive abstraction pattern it's now easy to provide drives that use finagle-http under the hood, or okhttp, or apache http, etc. Client writers can share their model and client code with helpful utilities without leaking dependencies. And most importantly, service owners can update dependencies and move to new scala versions without fearing that their dependencies are going to cause runtime or compile time issues against pulled in clients, all while still iterating quickly and safely.

