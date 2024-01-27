---
layout: post
title: Unit testing DNS failovers
date: 2016-06-02 23:17:06.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dns
- java
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558791887;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:7777;}i:1;a:1:{s:2:"id";i:4589;}i:2;a:1:{s:2:"id";i:4991;}}}}

permalink: "/2016/06/02/unit-testing-dns-failovers/"
---
Something that's come up a few times in my career is the difficulty of validating if and when your code can handle actual DNS changes. A lot of times testing that you have the right JVM settings and that your 3rd party clients can handle it involves mucking with hosts files, nameservers, or stuff like Route53 and waiting around. Then its hard to automate and deterministically reproduce. However, you can hook into the DNS resolution in the JVM to control what gets resolved to what. And this way you can tweak the resolution in a test and see what breaks! I found some info at this [blog post](rkuzmik.blogspot.com/2006/08/local-managed-dns-java_11.html) and cleaned it up a bit for usage in scala.

The magic sauce to pull this off is to make sure you override the default `sun.net.spi.nameservice.NameServiceDescriptor`. Internally in the `InetAddress` class it tries to load an instance of the interface `NameServiceDescriptor` using the Service loader mechanism. The service loader looks for resources in `META-INF/services/fully.qualified.classname.to.override` and instantiates whatever fully qualified class name is that class name override file.

For example, if we have

```
  
cat META-INF/services/sun.net.spi.nameservice.NameServiceDescriptor  
io.paradoxical.test.dns.LocalNameServerDescriptor  

```

Then the `io.paradoxical.test.dns.LocalNameServerDescriptor` will get created. Nice.

What does that class actually look like?

```scala
  
class LocalNameServerDescriptor extends NameServiceDescriptor {  
 override def getType: String = "dns"

override def createNameService(): NameService = {  
 new LocalNameServer()  
 }

override def getProviderName: String = LocalNameServer.dnsName  
}  

```

The type is of `dns` and the name service implementation is our own class. The provider name is something we have custom defined as well below:

```scala
  
object LocalNameServer {  
 Security.setProperty("networkaddress.cache.ttl", "0")

protected val cache = new ConcurrentHashMap[String, String]()

val dnsName = "local-dns"

def use(): Unit = {  
 System.setProperty("sun.net.spi.nameservice.provider.1", s"dns,${dnsName}")  
 }

def put(hostName: String, ip: String) = {  
 cache.put(hostName, ip)  
 }

def remove(hostName: String) = {  
 cache.remove(hostName)  
 }  
}

class LocalNameServer extends NameService {

import LocalNameServer.\_

val default = new DNSNameService()

override def lookupAllHostAddr(name: String): Array[InetAddress] = {  
 val ip = cache.get(name)  
 if (ip != null && !ip.isEmpty) {  
 InetAddress.getAllByName(ip)  
 } else {  
 default.lookupAllHostAddr(name)  
 }  
 }

override def getHostByAddr(bytes: Array[Byte]): String = {  
 default.getHostByAddr(bytes)  
 }  
}  

```

Pretty simple. We have a cache that is stored in a singleton companion object with some helper methods on it, and all we do is delegate looking into the cache. If we can resolve the data in the cache we return it, otherwise just proxy it to the default resolver.

The `use` method sets a system property that says to use the dns resolver of name `local-dns` as the highest priority `nameservice.provider.1` (lower numbers are higher priority)

Now we can write some tests and see if this works!

```scala
  
@RunWith(classOf[JUnitRunner])  
class DnsTests extends FlatSpec with Matchers {  
 LocalNameServer.use()

"DNS" should "resolve" in {  
 val google = resolve("www.google.com")

google.getHostAddress shouldNot be("127.0.0.1")  
 }

it should "be overridable" in {  
 LocalNameServer.put("www.google.com", "127.0.0.1")

val google = resolve("www.google.com")

google.getHostAddress should be("127.0.0.1")

LocalNameServer.remove("www.google.com")  
 }

it should "be undoable" in {  
 LocalNameServer.put("www.google.com", "127.0.0.1")

val google = resolve("www.google.com")

google.getHostAddress should be("127.0.0.1")

LocalNameServer.remove("www.google.com")

resolve("www.google.com").getHostAddress shouldNot be("127.0.0.1")  
 }

def resolve(name: String) = InetAddress.getByName(name)  
}  

```

Happy dns resolving!

