---
layout: post
title: Review of my first time experience with haskell editors
date: 2013-09-30 08:00:39.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Discussion
- Rants
tags:
- haskell
- ide
meta:
  _wpas_done_all: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561721293;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4327;}i:1;a:1:{s:2:"id";i:4725;}i:2;a:1:{s:2:"id";i:4262;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/09/30/time-experience-haskell-editors/"
---
When you start learning a new language the first hurdle to overcome is how to edit, compile, and debug an application. In my professional career I rely heavily on visual studio and intellij IDEA as my two IDE workhorses. Things just work with them. I use visual studio for C#, C++, and F# development and IDEA for everything else (including scala, typescript, javascript, sass, ruby, and python).

IDEA had a haskell plugin but it didn't work and caused exceptions in intellij using intellij 12+. Since my main ide's wouldn't work with haskell I took to researching what I could use.

## Requirements

While some people frown on the idea of an IDE, I personally like them. To quote [Erik Meijer](https://twitter.com/headinthebox/status/379847787619184640)

> I am hooked on autocomplete. When I type xs "DOT" it is a cry for help what to pick, map, filter, flatMap. Don't know upfront.

Not only that, but I want the build system hidden away, I want immediate type checking and error highlighting. I want code navigation, syntax highlighting, and an integrated debugger. I want all that and I don't want to have to spend more than 30 seconds getting started. The reason being is that I have problems to solve! The focus should be on the task at hand, not fiddling with an editor.

In college I used VIM and while it was excellent at what it did, I found that it really wasn't for me. Switching between the command mode and the edit mode was annoying, and I really just want to use the mouse sometimes. I also tried EMACS, and while it did the job, I think the learning curve was too high without enough "_oo! that's cool!_" moments to keep me going. If I did a lot of terminal work (especially remote) then mastering these tools is a must, but I don't. I know enough to do editing when I have to, but I don't want to develop applications in that environment. When you find a good IDE (whether its a souped up editor or not) your productivity level skyrockets.

## Getting Haskell working

Even though I'm on a windows machine I still like to use unix utilities. I have a collection of unix tools like ls, grep, sort, etc. Turns out this is kind of a problem when installing Haskell. You need to have the official Gnu Utils for wget, tar, and gzip otherwise certain installations won't work. Also if you have tortoise GIT installed on your machine and in your path, some other unix utils are also available. To get Haskell working properly I had to make sure the GNU utils were first in the path before any of the other tools.

On top of that, I wasn't able to get the cabal package for Hoogle to install on windows. About a week later, when I was trying to get Haskell up and running again I found [this post](https://code.google.com/p/ndmitchell/issues/detail?id=619) which mentioned that they had just fixed a windows build problem.

## Leksah

Once haskell was built, I turned to finding an IDE. My first google pointed me to Leksah, which at initially like exactly what I wanted. It had auto completion, error checking, debugging, etc. And it had a sizzlin dark theme that I thought was cool. I installed the 2013 Haskell platform (which contains GHC 7.6.3) and tried to run the Leksah build I got from their site. Being a Haskell novice, I didn't know that you had to run Leksah that is compiled against the GHC version you have, so nothing worked! Leksah loaded, but I was immediately bombared with questions about workspaces, cabal files, modules, etc. This was overwhelming. I just wanted to type in some haskell and run it.

Once I figured that all out though, I couldn't get the project to debug or any of the haskell modules to load. Auto complete also wouldn't work.

Frustrated, I spent 2 days searching for solutions. I eventually realized I needed the right version of Leksah and found a beta build posted on in the Leksah google forums. Unfortunately this had other issues. I again couldn't debug (clicking the debug button enabled and then immediately disabled), the GTK skin looked wonky, and right clicking opened menus 20 pixels above from where the mouse actually was.

Given all this, I gave up on Leksah.

## SublimeText

The next step was sublime text with the sublime text haskell plugin. I was skeptical here since sublime text is really just a fancy text editor, but people swore by it so I gave it a shot. Here I had better luck getting things to work, but I was still unhappy. For a person new to Haskell, the exploratory aspect just wasn't there. There's no integration with GHCi for debugging, and I couldn't search packages for what I wanted. Auto complete was faulty at best, it wouldn't pick up functions in other files and wouldn't prompt me half the time.

Still, it looked sharp and loaded fast. I was a big fan of the REPL plugin, but loading things into the REPL was kind of a pain. Also I liked all the hot keys, adding inferred types was easy, checking types was reasonably easy, but the lack of a good code navigation and proper auto completion irked me.

**EDIT:** I originally wrote this a few weeks ago even though it was just published today, and since then the REPL loading was fixed and so were a bunch of other bugs. In the end I've actually been using sublime text 2 for most of the small project editing, even though I liked the robustness of EclipseFP a lot.

## EclipseFP

EclipseFP is where I finally hit my stride. Almost immediately everything worked. Debugging was great, code navigation, syntax highlighting, code time errors, etc. Unfortunately I couldn't get the hoogle panel to work but the developer was incredibly responsive and worked me through the issue (and updated the plugin to work with the new eclipse version "Kepler"). I also enjoyed the fact that working in a file auto-loaded it into GHCi REPL so I could edit then test my functions quicker. On top of that, the developer recently submitted a pull request to the eclipse theme plugin so new dark themes will be available soon!

One thing I do wish is that the REPL had syntax highlighting like the sublimeText REPL did, but that's OK.

## Conclusion

In the end, while I can see how people more familiar with Haskell would choose the lightweight editor route (such as sublime), people new to the language really need a way to get up and running fast. Without that, it's easy to get turned off from trudging through and learning a new language. A good IDE helps a user explore and automates a lot of the boring nastiness that comes with real development.

