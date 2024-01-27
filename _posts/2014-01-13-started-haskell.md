---
layout: post
title: Getting started with haskell
date: 2014-01-13 08:00:10.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- haskell
- projects
- setup
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _syntaxhighlighter_encoded: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561000134;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4244;}i:1;a:1:{s:2:"id";i:4725;}i:2;a:1:{s:2:"id";i:4348;}}}}

permalink: "/2014/01/13/started-haskell/"
---
I wanted to share how I've finally settled on my haskell development environment and how I got it set up, since the process in the end wasn't that trivial. Hopefully anyone else starting in haskell can avoid the annoyances and pitfalls that I ran into and get up and running (and doing haskell) quickly.

## Getting haskell

First, download the haskell platform from [haskell.org](http://www.haskell.org/platform/). This is pretty easy. At this point you should have `ghc` (the compiler) and `ghci` (the interactive REPL) installed and in your path.

Aslo at this point you should have `cabal` installed. Cabal is haskells package manager. It's like ruby gems, or .NET nuget, or node's NPM (gah, so many!).

## Get sublime text

As much of a visual studio fanboy that I am, I have to say that using sublimetext for haskell has turned out to be really nice. Most of the haskellers I asked on twitter also use sublime text so it has a very supportive and active community. If you have sublime text already, install the [sublime haskell plugin](https://github.com/SublimeHaskell/SublimeHaskell) via sublimes package manager.

SublimeHaskell leverages command line haskell tools, like `hdevtools` and `ghc-mod` to give you language completion, documentation, type inference (and type completion). I highly recommend these, since it makes development a lot easier.

On windows you may run into issues that you can't install the `unix-2.7` package. That's OK. You don't need to install it from cygwin, contrary to lots of stack overflow answers. Instead, go to the following [fork](https://github.com/mvoidex/hdevtools) of the `hdevtools` project. Go to the project folder and do

```
  
\> cabal configure  
\> cabal build  
\> cabal install  

```

Basically this just builds the project from source, then installs it into your cabal package path (which is for your user). Hdevtools gives access to type information for files on the command line, so it's really important to get this working, otherwise your type inference in sublimetext won't work.

## Get sublime load file to REPL plugin

This [plugin](https://github.com/laughedelic/LoadFileToRepl) is really handy, it will auto load your module file into the REPL.

## If you have problems...

I had some issues getting all this to work on windows. Cabal would fail trying to install certain packages giving gzip errors. If you run into that problem, chances are you have conflicting mingw/cygwin or gnu utils in your path. I suggest removing them and getting the official gnu32 utils (which solves this problem). More info can be found at this [stack overflow question](http://stackoverflow.com/questions/7523151/hoogle-data-on-windows).

I also had issues where I had too many things in my path and the haskell installer (when adding itself to the path) pushed out some path items. Windows has a path length limit so if things get weird, just check that what you _think_ should be there actually is.

## Create a project

Creating a haskell project, for a novice, isn't trivial. You need to initialize a project directory with `cabal`, configure a bunch of crap in the `project.cabal` file, and then re-configure cabal. If you want to have unit testing set up with auto-discoverable tests (like you can do in Java or C# with test attributes), you have to one step further and create a seperate test runner and add a bunch of magic haskell preprocessor tags. I can never remember how to do any of this and I find it endlessly frustrating every time I have to make a new project. Some IDE's like EclipseFP automatically set all this up for you, but if you go the sublime route you're on your own.

However, I took the time to write a [grunt scaffolding task](https://github.com/devshorts/grunt-init-haskell-test) that will automatically create your cabalized project for you, AND set up your unit testing infrastructure so you don't have to think about it. For those not familiar, grunt is a javascript build/automation tool that leverages node. I went with grunt to write the scaffolding since I wanted to try something new, and it looked easy (I'm a sucker for easy things).

Once you check out the project and put it in your grunt home directory, you can type `grunt-init haskell-test` and it'll prompt you for a project name, create a cabal file and project folders (with src and test directories), create a test runner, and create an initial unit testing file with a sample test in it. After that it'll automatically run `cabal configure --enable-tests` and you are good to go! To compile it type `cabal build` and to run it type `cabal test` (or sublime text will do this all for you since the sublime plugin works with cabal created projects).

## Understanding the cabal file

A few things that got me when starting with haskell is understanding the cabal file. The cabal file is like a `.proj` file for f# or c#, or your maven `pom.xml` for Java. It describes dependencies, where the source directories are, etc. When you add dependencies to new libraries you'll need to update this file.

## Adding new tests

Unfortunately the grunt task I wrote only initializes the scaffold. It won't set up boilerpate for new unit tests. However, at this point just copy the unit test file you have, update the unit test main (`TestMain.hs`) to import the new unit test fixture module, and everything will be cool. Note, using HTF there are some conditions. Functions that are prefixed with `test_` must be of type `Assertion`, there needs to be a special preprocessor macro at the top of the file, quickcheck functions are prefixed with `prop_`, etc. All of that is listed [here](http://hackage.haskell.org/package/HTF-0.5.0.0/docs/Test-Framework-Tutorial.html), and everything is already set up with the grunt scaffold.

## Conclusion

At this point you should be up and running with syntax highlighting, type inference, syntax completion, error highlighting, linting, unit testing, REPL, and debuggging (via the REPL). All things that you want from a full fledged programming experience. I'm not going to lie, visual studio is still an infinitely more pleasurable experience, BUT once the main project boilerplate was removed working in haskell is much more enjoyable.

