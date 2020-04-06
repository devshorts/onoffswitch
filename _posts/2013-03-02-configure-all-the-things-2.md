---
layout: post
title: Configure all the things
date: 2013-03-02 16:46:50.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- Best Practices
- c#
- configuration
- Rx
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1555170277;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1043;}i:1;a:1:{s:2:"id";i:4862;}i:2;a:1:{s:2:"id";i:4631;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/02/configure-all-the-things-2/"
---
I personally think that just about everything should be configurable, unless it's absolutely never going to change. Even then, make it configurable, because it may change in the future. Think about your favorite command line tools, and the extensibility they have. They're powerful because they are dynamic. They can be configured for a myriad of options and scenarios.

Configuration doesn't just imply things that are public to users, like the background color or autosave frequency, but it can expose non-compile time settings that help you tune your application to unknown environments. Most software can't possibly be tested in every environment that it will be used. However, if you make your code flexible and configurable, you can minimize damage and buy yourself some time when things go bad. You can possibly even find a configuration that works around the problem. Configuration can improve someone's experience by modifying assumptions you made while developing with a toggle or tweak of a configuration value.

## What to configure?

Some examples of things that should be configurable (other than outward user options):

- **Timeouts**. Any time you are setting timeouts for something, you should make this configurable. Maybe that 5-minute timeout you thought was impossible is actually happening, and you need to make it 6 minutes, at least temporarily
- **Retries**. If you are going to retry something a certain number of times, you should make this value configurable. You should also make the interval of the retry configurable.
- **Thresholds**. If something has a threshold, it should be configurable. You should be able to tighten the threshold or loosen it
- **Max and mins**. Anything that has an upper or lower limit should be configurable.
- **Optional UI items**. Some parts of an application aren't used by everyone. Even if you never plan on doing it, make it toggleable. This way you can toggle the visibility (and hopefully loading complexity) of a widget or piece of the application that isn't useful to a certain user or user demographic. Some clients may not like item XYZ always showing up, even if you think its a core part of the application. If you can just disable it with ease, then that makes them happy not to see it. This keeps the codebase clean (less custom branches), your support team happy, and you can keep being lazy and read reddit or whatever because you don't need to make a custom build (which of course requires unit testing, QA review, deployment, etc), which makes you happy.
- **Things that start at runtime**. Code internals should be able to be toggled. If you have a thread that spins up on startup, and is suddenly [borking](http://onoffswitch.net/wp-content/uploads/2013/03/swedishchef.jpg) everything, then you can have a way to temporarily turn it off. This can save you, your support team, and the client a big headache while you find out why its not working. More than once I've run across something unexpected that happened, resulting in a runaway thread or thrashing disk. Wrapping everything in a configuration toggle lets you turn it off, and safely bypass an entire swath of the application. I've set it up before that for all items that run at startup there is a way to dynamically disable that option in a config. A config block like `DisabledRunTimeThreadNamePrefixes` means I can tack in a semicolon seperated list of known thread name prefixes (always name your threads!) and on startup those threads won't run. This is obviously not a long term solution, but does help with debugging if threads aren't playing nice or being destructive. While defensive programming can go a long way, sometimes you need to immediately turn off an entire section and not let it hit your code.
- **Execution of 3rd party libraries/programs**. Imagine you do a netstat periodially, to see who is connected to what, as part of an application health monitor. However, netstat is choking on the network, and delaying other health statistics or waiting threads. You should make sure to wrap this kind of execution in a configuration, so that you can turn it on or off, depending on the scenario. 
- **Optional features**. Sometimes we have pet projects, and we love working on them, but nobody asked for it. If it works, the client may think this is the best thing ever. If it doesn't work, they really don't want it screwing up core requirements. These kinds of things should all be configurable, so you can turn them off if desired.
- **Paths**. If you can configure where things go you should. This makes it easy to change where you log or put temporary files, etc and makes it easy to modify your application in unknown environments. It's best to try not to hard-code values anywhere.
- **Late in release additions**. If you're releasing tomorrow, and some important feature just came in and HAS to be done (scope creep is its own issue...), wrap that feature set in a configuration. Late in the game features are more likely to have bugs especially related to interaction of other parts in the system. This is just because they haven't had the same QA time to be vetted. The last thing you want is to add a feature (or non-critical bug fix), think it's cool, then push it 1000+ clients only to have it break. Trust me.

## Getting at a config

While taking configuration via the command line is a simple and flexible thing to do, it doesn't scale well and can be difficult to maintain. There are a lot of ways to manage configurations. You can use an app.config, or a web.config (if its a website). You can store key value pairs like properties files do, or you can store your configuration in an xml file. Personally, I find that a known XML file is a great way to store a config. It's easy to share, read, serialize and deserialize.

If you decide to go the XML route for your configuration, and you're using C#, you can use the old style `XmlSerializer` or the new `DataContractSerializer`. The XmlSerializer works well for configurations because it uses an "opt out" paradigm for serializing items. This means anything not marked with `XmlIgnore` gets serialized. The DataContractSerializer is the other way around. It is "opt in", so for every field you want serialized you have to mark it with `DataMember`. Technically DataContractSerializer is faster (and has other advantages), but when dealing with configs you typically aren't serializing and deserializing enormous configs thousands of times over. Also configs tend to be relatively simple constructs so you don't need much of the fancy stuff that comes with DataContractSerializer. I like using `XmlSerializer` because it's easy.

A neat feature of XmlSerializer is the [`DefaultValue`](http://www.distribucon.com/blog/MarkingDefaultValuesToControlXMLSerialization.aspx) attribute, which lets you tag members on a serializable class with what their default value should be. When serializing a class, the serializer will only write values if the property is anything _other_ than the default. This way, you can load and save your configs without exposing the internals of your configuration class. The upside of this is that your configs stay clean, and you can easily see what has changed from the default. The downside is that if you plan on exposing these options, then they won't show up by default. If you go this route, it would be worth keeping a master list of configuration options, where they are located (which config file, xml block, etc), what the default value is, what the option does, and why it exists.

When loading the config, make sure that your config deserialization is exception safe. If the config is corrupt, log that you couldn't' load elements and then use the default settings (unless it's mission critical to have the config settings and then it's better to fail fast).

It's also worth thinking about the situation where configs can't be found. I like to make sure that my config loading code is forgiving. It will look in a set of known folders, and move up the local hierarchy until it can find something. If it finds something it'll make sure to log _which_ config it loaded. This way if there are config conflicts (for some reason if I have multiple configs floating around) I can see which config it loaded. Depending on your application architecture, this can be a reasonable solution. If you have a multi-process application, you can design it so that there is only one config for everyone. This makes maintaining configuration options easier, since it's all in one file. If you still can't find the configs, log an exception or a warning and then resort to the default values. Don't create a hard dependency on configuration, if you can avoid it. This makes re-using code that relies on configurations easier, since you don't need to have a configuration file to use it.

## Live loading with filewatchers and Rx

It's also pretty easy to live-load your configurations, instead of just once on startup. If you create a file-watcher on your config class, and re-load it when it's changed, then you can have live up-to-date configurations. If your config file is live-editable, you can even throttle reloading the config at some regular intervals using [Rx](http://msdn.microsoft.com/en-us/data/gg577609.aspx), so you don't spam your system with configuration reloading. Just make sure to reference all your configuration options directly from the config class and don't make local copies of config values. Below is a simple example that will call a reload config function in 2 second intervals, while a file is being edited.

[csharp]  
private void InitFileWatcher()  
{  
 \_watcher = new FileSystemWatcher(Path.GetDirectoryName(Path.GetFullPath(Config.Path)), Path.GetFileName(Config.Path));  
 Observable.FromEventPattern\<FileSystemEventHandler, FileSystemEventArgs\>(  
 ev =\> \_watcher.Changed += ev,  
 ev =\> \_watcher.Changed -= ev)  
 .Sample(TimeSpan.FromSeconds(Config.Instance.ConfigReloadTime)).Subscribe(ReloadConfig);

\_watcher.NotifyFilter = NotifyFilters.LastWrite;  
 \_watcher.EnableRaisingEvents = true;  
}

private void ReloadLogConfig(EventPattern\<FileSystemEventArgs\> obj)  
{  
 Log.Debug(this, "{0} path was changed ({1}), reloading config", obj.EventArgs.FullPath, obj.EventArgs.ChangeType);  
}  
[/csharp]

## Conclusion

I think configuration is just one part of defensive and flexible programming. Not all configurable items are about turning something on or off or avoiding bugs. They should also be about performance-tuning and optimization. In the end, if you aren't sure, make it configurable. You'll find that the worst that happens when you do is that you don't ever use that configuration value.

[![](http://onoffswitch.net/wp-content/uploads/2013/03/configureAllThethings..png)](http://onoffswitch.net/wp-content/uploads/2013/03/configureAllThethings..png)

