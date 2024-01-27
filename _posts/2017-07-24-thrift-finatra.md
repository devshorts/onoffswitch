---
layout: post
title: From Thrift to Finatra
date: 2017-07-24 23:36:12.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Cross Post
tags:
- finatra
- http
- scala
- services
- thrift
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554367279;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4919;}i:1;a:1:{s:2:"id";i:4945;}i:2;a:1:{s:2:"id";i:4991;}}}}
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'

permalink: "/2017/07/24/thrift-finatra/"
---
> Originally posted on the [curalate engineering blog](http://engineering.curalate.com/2017/07/05/from-thrift-to-finatra.html)

There are a million and one ways to do (micro-)services, each with a million and one pitfalls. At Curalate, we've been on a long journey of splitting out our monolith into composable and simple services. It's never easy, as there are a lot of advantages to having a monolith. Things like refactoring, code-reuse, deployment, versioning, rollbacks, are all atomic in a monolith. But there are a lot of disadvantages as well. Monoliths encourage poor factoring, bugs in one part of the codebase force rollbacks/changes of the entire application, reasoning about the application in general becomes difficult, build times are slow, transient build errors increase, etc.

To that end our first foray into services was built on top of Twitter [Finagle stack](https://twitter.github.io/finagle/). If you go to the page and can't figure out what _exactly_ finagle does, I don't blame you. The documentation is lackluster and in and of itself is quite low-level. Finagle defines a service as a function that transforms a request into a response, and composes services with filters that manipulate requests/responses themselves. It's a clean abstraction, given that this is basically what all web service frameworks do.

# Thrift

Finagle by itself isn't super opinionated. It gives you building blocks to build services (service discovery, circuit breaking, monitoring/metrics, varying protocols, etc) but doesn't give you much else. Our first set of services built on finagle used Thrift over HTTP. [Thrift](https://Thrift.apache.org/), similiar to protobuf, is an intermediate declarative language that creates RPC style services. For example:

```text
  
namespace java tutorial  
namespace py tutorial

typedef i32 int // We can use typedef to get pretty names for the types we are using  
service MultiplicationService  
{  
 int multiply(1:int n1, 2:int n2),  
}  

```

Will create an RPC service called `MultiplicationService` that takes 2 parameters. Our implementation at Curalate hosted Thrift over HTTP (serializing Thrift as JSON) since all our services are web based behind ELB's in AWS.

We have a lot of services at Curalate that use Thrift, but we've found a few shortcomings:

## Model Reuse

Thrift forces you to use primitives when defining service contracts, which makes it difficult to share lightweight models (with potentially useful utilities) to consumers. We've ended up doing a lot of mapping between generated Thrift types and shared model types. Curalate's backend services are all written in Scala, so we don't have the same issues that a company like Facebook (who invented Thrift) may have with varying languages needing easy access to RPC.

## Requiring a client

Many times you want to be able to interact with a service without needing access to a client. Needing a client has made developers to get used to cloning service repositories, building the entire service, then entering a Scala REPL in order to interact with a service. As our service surface area expands, it's not always feasible to expect one developer to build another developers service (conflicting java versions, missing SBT/Maven dependencies or settings, etc). The client requirement has led to services taking heavyweight dependencies on other services and leaking dependencies. While Thrift doesn't force you to do this, this has been a side effect of it taking extra love and care to generate a Thrift client properly, either by distributing Thrift files in a jar or otherwise.

## Over the wire inspection

With Thrift-over-HTTP, inspecting requests is difficult. This is due to the fact that these services use Thrift serialization, which unlike JSON, isn't human-readable.

Because Thrift over HTTP is all POSTs to `/`, tracing access and investigating ELB logs becomes a jumbled mess of trying to correlate times and IP's to other parts of our logging infrastructure. The POST issue is frustrating, because it's impossible for us to do any semantic smart caching, such as being able to insert caches at the serving layer for retrieval calls. In a pure HTTP world, we could insert a cache for heavily used GETs given a GET is idempotent.

## RPC API design

Regardless of Thrift, RPC encourages poorly unified API's with lots of specific endpoints that don't always jive. We have many services that have method topologies that are poorly composable. A well designed API, and cluster of API's, should gently guide you to getting the data you need. In an ideal world if you get an ID in a payload response for a data object, there should be an endpoint to get more information about that ID. However, in the RPC world we end up with a batch call here, a specific RPC call there, sometimes requiring stitching several calls to get data that should have been a simple domain level call.

## Internal vs External service writing

We have lot of public REST API's and they are written using the Lift framework (some of our oldest code). Developers moving from internal to external services have to shift paradigms and move from writing REST with JSON to RPC with Thrift.

Overall Thrift is a great piece of technology, but after using it for a year we found that it's not necessarily for us. All of these things have prompted a shift to writing REST style services.

# Finatra

[Finatra](https://twitter.github.io/Finatra/) is an HTTP API framework built on top of Finagle. Because it's still Finagle, we haven't lost any of our operational knowledge of the underlying framework, but instead we can now write lightweight HTTP API's with JSON.

With Finatra, all our new services have [Swagger](http://swagger.io/) automatically enabled so API exploration is simple. And since it's just plain JSON using [Postman](https://www.getpostman.com/) is now possible to debug and inspect APIs (as well as viewing requests in [Charles](https://www.charlesproxy.com/) or other proxies).

With REST we can still distribute lightweight clients, or more importantly, if there are dependency conflicts a service consumer can very quickly roll an HTTP client to a service. Our ELB logs now make sense and our new API's are unified in their verbiage (GET vs POST vs PUT vs DELETE) and if we _want_ to write RPC for a particular service we still _can_.

There are a few other things we like about Finatra. For those developers coming from a background of writing HTTP services, Finatra feels familiar with the concept of controllers, filters, unified test-bed for spinning up build verification tests (local in memory servers), dependency injection (via [Guice](https://github.com/google/guice)) baked in, sane serialization using Jackson, etc. It's hard to do the wrong thing given that it builds strong production level opinions onto Finagle. And thankfully those opinions are ones we share at Curalate!

We're not in bad company -- Twitter, [Duolingo](http://making.duolingo.com/rewriting-duolingos-engine-in-scala), and others are using Finatra in production.

