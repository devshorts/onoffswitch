---
layout: post
title: Logging the easy way
date: 2016-01-28 21:23:02.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- godaddy
- logging
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560268662;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4586;}i:1;a:1:{s:2:"id";i:4945;}i:2;a:1:{s:2:"id";i:4627;}}}}

permalink: "/2016/01/28/logging-easy/"
---
> This is a cross post from [the original posting at godaddy's engineering blog](http://engineering.godaddy.com/logging-the-easy-way/). This is a project I have spent considerable time working on and leverage a lot.

Logging is a funny thing. Everyone knows what logs are and everyone knows you should log, but there are no hard and fast rules on how to log or what to log. Your logs are your first line of defense against figuring out issues live. Sometimes logs are the only line of defense (especially in time sensitive systems).

That said, in any application good logging is critical. Debugging an issue can be made ten times easier with simple, consistent logging. Inconsistent or poor logging can actually make it impossible to figure out what went wrong in certain situations. Here at GoDaddy we want to make sure that we encourage logging that is consistent, informative, and easy to search.

Enter the GoDaddy Logger. This is a SLF4J wrapper library that encourages us to fall into the pit of success when dealing with our logging formats and styles in a few ways:

- Frees you from having to think about what context fields need to be logged and removes any worries about forgetting to log a value,
- Provides the ability to skip personal identifiable information from being logged,
- Abstracts out the actual format of the logs from the production of them. By decoupling the output of the framework from the log statements themselves, you can easily swap out the formatter when you want to change the structure and all of your logging statements will be consistently logged using the new format.

A lot of teams at GoDaddy use ELK (Elasticsearch, Logstash, Kibana) to search logs in a distributed system. By combining consistent logging with ELK (or Splunk or some other solution), it becomes relatively straight forward for developers to correlate and locate related events in their distributed systems.

# THE GODADDY LOGGER

In an effort to make doing the right thing the easy thing, our team set out to build an extra layer on top of SLF4J – The GoDaddy Logger. While SLF4J is meant to abstract logging libraries and gives you a basic logging interface, our goal was to extend that interface to provide for consistent logging formats. One of the most important things for us was that we wanted to provide an easy way to log objects rather than having to use string formatting everywhere.

# CAPTURING THE CONTEXT

One of the first things we did was expose what we call the ‘with’ syntax. The ‘with’ syntax builds a formatted key value pair, which by default is “key=value;”, and allows logging statements to be more human readable. For example:

```java
  
logger.with(“first-name”, “GoDaddy”)  
 .with(“last-name”, “Developers!”)  
 .info(“Logging is fun”);  

```

Using the default logging formatter this log statement outputs:

Logging is fun; first-name=“GoDaddy”; last-name=”Developers!”.  
We can build on this to support deep object logging as well. A good example is to log the entire object from an incoming request. Instead of relying on the .toString() of the object to be its loggable representation, we can crawl the object using reflectasm and format it globally and consistently. Let’s look at an example of how a full object is logged.

```java
  
Logger logger = LoggerFactory.getLogger(LoggerTest.class);  
Car car = new Car(“911”, 2015, “Porsche”, 70000.00, Country.GERMANY, new Engine(“V12”));  
logger.with(car).info(“Logging Car”);  

```

Like the initial string ‘with’ example, the above log line produces:

```java
  
14:31:03.943 [main] INFO com.godaddy.logger.LoggerTest – Logging Car; cost=70000.0; country=GERMANY; engine.name=”V12”; make=”Porsche”; model=”911”; year=2015  

```

All of the car objects info is cleanly logged in a consistent way. We can easily search for a model property in our logs and we won’t be at the whim of spelling errors of forgetful developers. You can also see that our logger nests object properties in dot object notation like “engine.name=”V12””. To accomplish the same behavior using SLF4J, we would need to do something akin to the following:

Use the Car’s toString functionality:

Implement the Car object’s toString function:

```java
  
String toString() {  
 Return “cost=” + cost + “; country=” + country + “; engine.name=” + (engine == null ? “null” : engine.getName()) … etc.  
}  

```

Log the car via it’s toString() function:

```java
  
logger.info(“Logging Car; {}”, car.toString());  

```

Use String formatting

```java
  
logger.info("Logging Car; cost={}; country={};e.name=\"{}\"; make=\"{}\"; model=\"{}\"; " + "year={}; test=\"{}\"", car.getCost(), car.getCountry(), car.getEngine() == null ? null : car.getEngine().getName(), car.getMake(), car.getModel(), car.getYear());  

```

Our logger combats these unfortunate scenarios and many others by allowing you to set the recursive logging level, which defines the amount of levels deep into a nested object you want to have logged and takes into account object cycles so there isn’t infinite recursion.

# SKIPPING SENSITIVE INFORMATION

The GoDaddy Logger provides annotation based logging scope support giving you the ability to prevent fields/methods from being logged with the use of annotations. If you don’t want to skip the entity completely, but would rather provide a hashed value, you can use an injectable hash processor to hash the values that are to be logged. Hashing a value can be useful since you may want to log a piece of data consistently but you may not want to log the actual data value. For example:

```java
  
import lombok.Data;  
  @Data  
 public class AnnotatedObject {  
 private String notAnnotated;  
   @LoggingScope(scope = Scope.SKIP)  
 private String annotatedLogSkip;  
 public String getNotAnnotatedMethod() {  
 return "Not Annotated";  
 }    
 @LoggingScope(scope = Scope.SKIP)  
 public String getAnnotatedLogSkipMethod() {  
 return "Annotated";  
 }  
   @LoggingScope(scope = Scope.HASH)  
 public String getCreditCardNumber() {  
 return "1234-5678-9123-4567";  
 }  
}  

```

If we were to log this object:

```java
  
AnnotatedObject annotatedObject = new AnnotatedObject();  
annotatedObject.setAnnotatedLogSkip(“SKIP ME”);  
annotatedObject.setNotAnnotated(“NOT ANNOTATED”);   
logger.with(annotatedObject).info(“Annotation Logging”);  

```

The following would be output to the logs:

```java
  
09:43:13.306 [main] INFO com.godaddy.logging.LoggerTest – Annotating Logging; creditCardNumber=”5d4e923fe014cb34f4c7ed17b82d6c58; notAnnotated=”NOT ANNOTATED”; notAnnotatedMethod=”Not Annotated”  

```

Notice that the annotatedLogSkip value of “SKIP ME” is not logged. You can also see that the credit card number has been hashed. The GoDaddy Logger uses Guava’s MD5 hashing algorithm by default which is not cryptographically secure, but definitely fast. And you’re able to provide your own hashing algorithm when configuring the logger.

# LOGGING CONTEXT

One of the more powerful things of the logger is that the ‘with’ syntax returns a new immutable captured logger. This means you can do something like this:

```java
  
Logger contextLogger = logger.with(“request-id”, 123);  
contextLogger.info(“enter”);   
// .. Do Work   
contextLogger.info(“exist”);  

```

All logs generated off the captured logger will include the captured with statements. This lets you factor out common logging statements and cleans up your logs so you see what you really care about (and make less mistakes).

# CONCLUSION

With consistent logging we can easily search through our logs and debug complicated issues with confidence. As an added bonus, since our log formatting is centralized and abstracted, we can also make team-wide or company-wide formatting shifts without impacting developers or existing code bases.

Logging is hard. There is a fine line between logging too much and too little. Logging is also best done while you write code vs. as an afterthought. We’ve really enjoyed using the GoDaddy Logger and it’s really made logging into a simple and unobtrusive task. We hope you take a look and if you find it useful for yourself or your team let us know!

For more information about the GoDaddy Logger, check out the [GitHub project](https://github.com/godaddy/godaddy-logger), or if you’re interested in working on these and other fun problems with us, check out our jobs page.

