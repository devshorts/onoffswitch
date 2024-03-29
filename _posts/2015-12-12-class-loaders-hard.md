---
layout: post
title: Plugin class loaders are hard
date: 2015-12-12 01:31:34.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- asm
- classloader
- java
- osgi
- plugin
- runtime
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561588999;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4844;}i:1;a:1:{s:2:"id";i:4306;}i:2;a:1:{s:2:"id";i:4699;}}}}

permalink: "/2015/12/12/class-loaders-hard/"
---
Plugin based systems are really common. Jenkins, Jira, wordpress, whatever. Recently I built a plugin workflow for a system at work and have been mired in the joys of the class loader. For the uninitiated, a class in Java is identified uniquely by the class loader instance it is created from as well as its fully qualified class name. This means that `foo.bar` class loaded by class loader A is _not the same_ as `foo.bar` class loaded by class loader B.

There are actually some cool things you can do with this, especially in terms of code isolation. Imagine your plugins are bundled as shaded jars that contain all the internal dependencies. By leveraging class loaders you can isolate potentially conflicting versions of libraries from the host application and the plugin. But, in order to communicate to the host layer, you need a strict set of shared interfaces that the host layer always owns. When building the uber jar you exclude the host interfaces from being bundled (and all its transitive dependencies which in maven can be done by using scope `provided`). This means that they will always be loaded by the host.

In general, class loaders are heirarchical. They ask their parent if a class has been loaded, and if so returns that. In order to do plugins you need to invert that process. First look inside the uber-jar, and then if you can't find a class then look up.

An example can be found [here](http://tech.puredanger.com/2006/11/09/classloader/) and copied for the sake of internet completeness:

```java
  
import java.net.URL;  
import java.net.URLClassLoader;  
import java.net.URLStreamHandlerFactory;  
import java.util.UUID;

public class PostDelegationClassLoader extends URLClassLoader {

private final UUID id = UUID.randomUUID();

public PostDelegationClassLoader(URL[] urls, ClassLoader parent, URLStreamHandlerFactory factory) {  
 super(urls, parent, factory);  
 }

public PostDelegationClassLoader(URL[] urls, ClassLoader parent) {  
 super(urls, parent);  
 }

public PostDelegationClassLoader(URL[] urls) {  
 super(urls);  
 }

public PostDelegationClassLoader() {  
 super(new URL[0]);  
 }

@Override  
 public Class\<?\> loadClass(String name) throws ClassNotFoundException {  
 try (ThreadCurrentClassLoaderCapture capture = new ThreadCurrentClassLoaderCapture(this)) {  
 Class loadedClass = findLoadedClass(name);

// Nope, try to load it  
 if (loadedClass == null) {  
 try {  
 // Ignore parent delegation and just try to load locally  
 loadedClass = findClass(name);  
 }  
 catch (ClassNotFoundException e) {  
 // Swallow - does not exist locally  
 }

// If not found, just use the standard URLClassLoader (which follows normal parent delegation)  
 if (loadedClass == null) {  
 // throws ClassNotFoundException if not found in delegation hierarchy at all  
 loadedClass = super.loadClass(name);  
 }  
 }  
 return loadedClass;  
 }  
 }

@Override  
 public URL getResource(final String name) {  
 final URL resource = findResource(name);

if (resource != null) {  
 return resource;  
 }

return super.getResource(name);  
 }  
}  

```

But this is just the tip of the fun iceberg. If all your libraries play nice then you may not notice anything. But I recently noticed using the apache xml-rpc library that I would get a SAXParserFactory class def not found exception, specifically bitching about instantiating the sax parser factory. I'm not the only one apparenlty, [here is a discussion](https://answers.atlassian.com/questions/104121/im-blocked-help-cannot-be-cast-to-javax.xml.parsers.saxparserfactory) about a JIRA plugin that wasn't happy. After much code digging I found that the classloader being used was the one bound to the threads current context.

Why in the world is there a classloader bound to thread local? JavaWorld has a [nice blurb](http://www.javaworld.com/article/2077344/core-java/find-a-way-out-of-the-classloader-maze.html:) about this

> Why do thread context classloaders exist in the first place? They were introduced in J2SE without much fanfare. A certain lack of proper guidance and documentation from Sun Microsystems likely explains why many developers find them confusing.
> 
> In truth, context classloaders provide a back door around the classloading delegation scheme also introduced in J2SE. Normally, all classloaders in a JVM are organized in a hierarchy such that every classloader (except for the primordial classloader that bootstraps the entire JVM) has a single parent. When asked to load a class, every compliant classloader is expected to delegate loading to its parent first and attempt to define the class only if the parent fails.
> 
> Sometimes this orderly arrangement does not work, usually when some JVM core code must dynamically load resources provided by application developers. Take JNDI for instance: its guts are implemented by bootstrap classes in rt.jar (starting with J2SE 1.3), but these core JNDI classes may load JNDI providers implemented by independent vendors and potentially deployed in the application's -classpath. This scenario calls for a parent classloader (the primordial one in this case) to load a class visible to one of its child classloaders (the system one, for example). Normal J2SE delegation does not work, and the workaround is to make the core JNDI classes use thread context loaders, thus effectively "tunneling" through the classloader hierarchy in the direction opposite to the proper delegation.

This means that whenever I'm delegating work to my plugins I need to be smart about capturing my custom plugin class loader and putting it on the current thread before execution. Otherwise if a misbehaving library accesses the thread classloader, it can now have access to the ambient root class loader and IFF the same class name exists in the host application it will load it. This could potentially conflict with other classes from the same package that aren't loaded this way and in general cause mayhem.

The solution here was a simple class modeled after .NET's disposable pattern using Java's try/finally auto closeable.

```java
  
public class ThreadCurrentClassLoaderCapture implements AutoCloseable {  
 final ClassLoader originalClassLoader;

public ThreadCurrentClassLoaderCapture(final ClassLoader newClassLoader) {  
 originalClassLoader = Thread.currentThread().getContextClassLoader();

Thread.currentThread().setContextClassLoader(newClassLoader);  
 }

@Override  
 public void close() {  
 Thread.currentThread().setContextClassLoader(originalClassLoader);  
 }  
}  

```

Which is used before each and every invocation into the interface of the plugin (where `connection` is the plugin reference)

```java
  
@Override  
public void start() throws Exception {  
 captureClassLoader(connection::start);  
}

@Override  
public void stop() throws Exception {  
 captureClassLoader(connection::stop);  
}

@Override  
public void heartbeat() throws Exception {  
 captureClassLoader(connection::heartbeat);  
}

private void captureClassLoader(ExceptionRunnable runner) throws Exception {  
 try (ThreadCurrentClassLoaderCapture capture = new ThreadCurrentClassLoaderCapture(connection.getClass().getClassLoader())) {  
 runner.run();  
 }  
}  

```

However, this isn't the only issue. Imagine a scenario where you support both class path loaded plugins AND remote loaded plugins (via shaded uber-jar). And lets pretend that on the classpath is a jar with the same namespaces and classes as that in an uberjar. To be more succinct, you have a delay loaded shared library on the class path, and a version of that library that is shaded loaded via the plugin mechanism.

Technically there shouldn't be any issues here. The class path plugin gets all its classes resolved from the root scope. The plugin gets its classes (of the same name) from the delegated provider. Both use the same shared set of interfaces of the host. The issue arrises if you have a library like reflectasm, which dynamically emits bytecode at runtime.

Look at this code:

```java
  
AccessClassLoader loader = AccessClassLoader.get(type);  
synchronized (loader) {  
 try {  
 accessClass = loader.loadClass(accessClassName);  
 } catch (ClassNotFoundException ignored) {  
 String accessClassNameInternal = accessClassName.replace('.', '/');  
 String classNameInternal = className.replace('.', '/');  
 ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE\_MAXS);  

```

Which is a snippet from reflectasm as its generating a runtime byte code emitter that can access fields for you. It creates a class name like `your.class.nameMethodAccess`. If the class name isn't found, it generates the bytecode and then writes it into the owning classes class loader.

In the scenario of a plugin using this library, it will check the loader and see that the plugin classloader AND rootscope loader do not have the emitted class name, and so a class not found exception is thrown. It will then write the class into the _target types class loader_. This would be the delegated loader, and provides the isolation we want.

However, if the class path plugin (what I call an embedded plugin) runs this code, the dynamic runtime class is written into the _root scope_ loader. This means that all delegating class loaders will eventually find this type since they always do a delegated pass to the root!

The important thing to note here is that using a delegated loader does not mean every class that comes out of it is tied to the delegated loader. Only classes that are found inside of the delegated loader are bound to it. If a class is resolved by the parent, the class is linked to the parent.

In this scenario with the root class loader being polluted with the same class name, I don't think there is much you can do other than avoid it.

Anyways, maybe I should have used OSGi...?

