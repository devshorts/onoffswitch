---
layout: post
title: Strongly typed http headers in finatra
date: 2017-02-12 00:31:26.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- finatra
- scala
meta:
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558278890;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4945;}i:1;a:1:{s:2:"id";i:4919;}i:2;a:1:{s:2:"id";i:4939;}}}}
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpcom_is_markdown: '1'
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'
  _wp_old_slug: passing-request-context-finatra

permalink: "/2017/02/12/stronlgy-typed-headers-finatra/"
---
When building service architectures one thing you need to solve is how to pass context between services. This is usually stuff like request id's and other tracing information (maybe you use [zipkin](http://zipkin.io/)) between service calls. This means that if you set request id FooBar123 on an entrypoint to service A, if service A calls service B it should know that the request id is still FooBar123. The bigger challenge is usually making sure that all thread locals keep this around (and across futures/execution contexts), but before you attempt that you need to get it into the system in the first place.

I'm working in [finatra](https://twitter.github.io/finatra/) these days, and I love this framework. It's got all the things I loved from [dropwizard](https://github.com/dropwizard/dropwizard) but in a scala first way. Todays challenge was that I wanted to be able to pass request http headers around between services in a typesafe way that could be used in thread local request contexts. Basically I want to send

```
  
X-Magic-Header someValue  

```

And be able to resolve that into a `MagicHeader(value: T)` class.

The first attempt is easy, just parse header values into case classes:

```scala
  
case class MagicHeader(value: String)  

```

But the question I have is how do I enforce that the header string `X-Magic-Value` is directly correlated to the case class `MagicHeader`?

```scala
  
object MagicHeader {  
 val key = "X-Magic-Header"  
}

case class MagicHeader(value: String)  

```

Maybe, but still, when someone sends the value out, they can make a mistake:

```scala
  
setRequestHeader("X-mag1c-whatevzer" -\> magicHeader.value)  

```

That sucks, I don't want that. I want it strictly paired. I'm looking for what is in essence a case class that has 2 fields: key, value, but where the key is _fixed_. How do I do that?

I like to start with how I want to use something, and then work backwards to how to make that happen. Given that, lets say we want an api kind of like:

```scala
  
object Experimental {  
 val key = "Experimental"

override type Value = String  
}  

```

And I'd like to be able to do something like

```scala
  
val experimentKey = Experimental("experiment abc")  
(experimentKey.key -\> experimentKey.value) shouldEqual  
 ("Experimental" -\> "experiment abc")  

```

I know this means I need an apply method somewhere, and I know that I want a tuple of (key, value). I also know that because I have a path dependent type of the second value, that I can do something with that

Maybe I can fake an apply method to be like

```scala
  
trait ContextKey {  
 val key: String

/\*\*  
 \* The custom type of this key  
 \*/  
 type Value

/\*\*  
 \* A tupel of (String, Value)  
 \*/  
 type Key = Product2[String, Value]

def apply(data: Value): Key = new Key {  
 override def \_1: String = key

override def \_2: Value = data  
 }  
}  

```

And update my object to be

```scala
  
object Experimental extends ContextKey {  
 val key = "Experimental"

override type Value = String  
}  

```

Now my object has a mixin of an apply method that creates an anonmyous tuple of type `String, Value`. You can create instances of `Experimental` but you can't ever set the key name itself! However, I can still _access_ the pinned key because the anonymous tuple has it!

But in the case that I wanted, I wanted to use these as http header values. Which means I need to be able to parse a string into a type of `ContextKey#Value` which is path dependent on the object type.

We can do that by adding now a few extra methods on the ContextKey trait:

```scala
  
trait ContextKeyType[T] extends Product2[String, T] {  
 def unparse: String  
}

trait ContextKey {  
 self =\>  
 val key: String

/\*\*  
 \* The custom type of this key  
 \*/  
 type Value

/\*\*  
 \* A tupel of (String, Value)  
 \*/  
 type Key = ContextKeyType[Value]

/\*\*  
 \* Utility to allow the container to provide a mapping from Value =\> String  
 \*  
 \* @param r  
 \* @return  
 \*/  
 def parse(r: String): Value

def unparse(v: Value): String

def apply(data: Value): Key = new Key {  
 override def \_1: String = key

override def \_2: Value = data

/\*\*  
 \* Allow a mapping of Value =\> String  
 \*  
 \* @return  
 \*/  
 override def unparse: String = self.unparse(data)

override def equals(obj: scala.Any): Boolean = {  
 canEqual(obj)  
 }

override def canEqual(that: Any): Boolean = {  
 that != null &&  
 that.isInstanceOf[ContextKeyType[\_]] &&  
 that.asInstanceOf[ContextKeyType[\_]].\_1 == key &&  
 that.asInstanceOf[ContextKeyType[\_]].\_2 == data  
 }  
 }  
}  

```

This introduces a parse and unparse method which converts things to and from strings. A http header object can now define how to convert it:

```scala
  
object Experimental extends ContextKey {  
 val key = "Experimental"  
 override type Value = String

override def parse(value: String): String = value

override def unparse(value: String): String = value  
}  

```

So, if we want to maybe send JSON in a header, or a long/int/uuid we can now parse and unparse that value pre and post wire.

Now lets add a utility to convert a `Map[String, String]` which could represent an http header map, into a set of strongly typed context values:

```scala
  
object ContextValue {  
 def find[T \<: ContextKey](search: T, map: Map[String, String]): Option[T#Value] = {  
 map.collectFirst {  
 case (key, value) if search.key == key =\> search.parse(value)  
 }  
 }  
}  

```

Back in finatra land, lets add a http filter

```scala
  
case class CurrentRequestContext(  
 experimentId: Option[Experimental.Value],  
)

object RequestContext {  
 private val requestType = Request.Schema.newField[CurrentRequestContext]

implicit class RequestContextSyntax(request: Request) {  
 def context: CurrentRequestContext = request.ctx(requestType)  
 }

private[filters] def set(request: Request): Unit = {  
 val data = CurrentRequestContext(  
 experimentId = ContextValue.find(Experimental, request.headerMap)  
 )

request.ctx.update(requestType, data)  
 }  
}

/\*\*  
 \* Set the remote context from requests  
 \*/  
class RemoteContextFilter extends SimpleFilter[Request, Response] {  
 override def apply(request: Request, service: Service[Request, Response]): Future[Response] = {  
 RequestContext.set(request)

service(request)  
 }  
}  

```

From here on out, we can provide a set of strongly typed values that are basically case classes with hidden keys

