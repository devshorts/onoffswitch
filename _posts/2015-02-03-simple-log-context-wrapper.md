---
layout: post
title: Simple log context wrapper
date: 2015-02-03 03:09:45.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- logging
- scala
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"f251743520edff2facd5c40ee081a536";a:2:{s:7:"expires";i:1558534014;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4593;}i:1;a:1:{s:2:"id";i:4862;}i:2;a:1:{s:2:"id";i:4945;}}}}

permalink: "/2015/02/03/simple-log-context-wrapper/"
---
I'm still toying around with the scala play! framework and I wanted to check out how I can make logging contextual information easy. In the past with .NET I've used and written libraries that wrap the current log provider and give you extra niceties with logging. One of my favorites was being able to do stuff like

```csharp
  
var foo = "1";  
var bar = "2";  
logger.With(new { foo, bar }).Info("data")  

```

Which would output a log line like

```
  
data, foo=1; bar=2  

```

The logger's "with" was even chainable so you could capture a previously built "with" context and re-use it. It was really nice when you want to create a baseline logging context for complex functions.

Java doesn't have the concept of anonymous classes and I'm not sure you can do optimizations with reflection like you can with .NET (creating reflective based property invokers into delegates).

Either way, this makes for a good experiment.

First off, the final product

```scala
  
val logInfo = With(  
 "request-uri" -\> rh.uri,  
 "request-time" -\> (System.currentTimeMillis() - start)  
)  
val compound = logInfo and With("date" -\> "bar")

logger.info("handled", compound)  

```

Which outputs

```
  
[info] LoggingFilter$ - handled request-uri=/; request-time=164; date=bar  
[info] LoggingFilter$ - handled request-uri=/assets/javascripts/jquery-1.9.0.min.js; request-time=347; date=bar  
[info] LoggingFilter$ - handled request-uri=/assets/stylesheets/main.css; request-time=362; date=bar  
[info] LoggingFilter$ - handled request-uri=/assets/images/favicon.png; request-time=478; date=bar  

```

This is a snippet I'm playing with in a root level timing filter for a scala play app. The "date -\> bar" association is just for demonstration of combining contexts

The full filter looks like

```scala
  
object LoggingFilter extends Filter{  
 val logger = Log(getClass)

val start = System.currentTimeMillis()

override def apply(f: (RequestHeader) =\> Future[Result])(rh: RequestHeader): Future[Result] = {

val result = f(rh)

val logInfo = With(  
 "request-uri" -\> rh.uri,  
 "request-time" -\> (System.currentTimeMillis() - start)  
 )

val compound = logInfo and With("date" -\> "bar")

logger.info("handled", compound)

result  
 }  
}  

```

Basically this creates a "With" object that is composable with other "with" objects which takes in a variable list of tuples and internally stores them as a map.

The factory function "Log" just instantiates the initial context object for tracking of state and captures the scala Play! logger to pass in

```scala
  
object Log {  
 def apply(src : Class[\_]) = new Log(Logger(src))  
}

class LogData(data: Map[String, String]) {

protected val logMap = data

def asLog = logMap.foldLeft("")((acc, kv) =\> acc + kv.\_1 + "=" + kv.\_2 + "; ").trim.stripSuffix(";")

def and(tup: (String, Any)) = {  
 val (x, y) = tup  
 new LogData(logMap.updated(x, y.toString))  
 }

def and(other: LogData) = {  
 new LogData(logMap ++ other.logMap)  
 }  
}  

```

Now you should see the log wrapper. It just wraps the scala play logger and takes in an extra log data if its passed in:

```scala
  
class Log(logger: LoggerLike) {

def getMessage(s: String, data: LogData): String = {  
 if (data != null) {  
 return s + " " + data.asLog  
 }

s  
 }

def info(s: String, m: LogData = null, t: Throwable = null) = logger.info(getMessage(s, m), t)

def debug(s: String, m: LogData = null, t: Throwable = null) = logger.debug(getMessage(s, m), t)

def warn(s: String, m: LogData = null, t: Throwable = null) = logger.warn(getMessage(s, m), t)

def error(s: String, m: LogData = null, t: Throwable = null) = logger.error(getMessage(s, m))

}  

```

At this point we just need to create the with context class, which again is just a factory function for the LogData class

```scala
  
object With {  
 def apply(tup: (String, Any)\*) = {  
 new LogData(tup.map(i =\> (i.\_1, i.\_2.toString)).toMap)  
 }  
}  

```

Since the main logger takes LogData instances this kind of works out well

Now we can get nicely uniform formatted messages for easy parsing in utilities like Splunk

