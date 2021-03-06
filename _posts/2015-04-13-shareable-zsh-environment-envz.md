---
layout: post
title: 'Shareable zsh environment: EnvZ'
date: 2015-04-13 01:44:26.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Discussion
tags:
- envz
- shell
- zsh
meta:
  _edit_last: '1'
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561704446;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4673;}i:2;a:1:{s:2:"id";i:4463;}}}}

permalink: "/2015/04/13/shareable-zsh-environment-envz/"
---
Introducing [EnvZ](https://github.com/devshorts/EnvZ).

## What is Envz?

During the course of normal production development we all tend to write a bunch of shell scripts and other useful command line utilities that help us out. Usually the end up being a bunch of one offs or stored in one mega .zshrc file. However, there's something to be said about having a small framework to share environment utilities and to use as a jump off to "version" a shared set of utilities with team mates.

With that in mind I've been building out a small zsh bootstrapper that builds on tools that I like and use (like yadr) and gives a way to add pluggable team modules into it. All a pluggable module is is a sym link to another directory that auto loads .sh files on shell start. And while it sounds like a small thing, it's actually really nice to be able to have different teams version different sets of shell scripts and be able to easily link and share them with environments.

## Example

For example, let me make a quick folder called `team1`, put in a dummy shell script and link it to my environment:

![Screen Shot 2015-04-12 at 7.36.45 PM](http://onoffswitch.net/wp-content/uploads/2015/04/Screen-Shot-2015-04-12-at-7.36.45-PM.png)

Notice how our function exists immediately after linking the env!

To unload it:

![Screen Shot 2015-04-12 at 7.38.18 PM](http://onoffswitch.net/wp-content/uploads/2015/04/Screen-Shot-2015-04-12-at-7.38.18-PM.png)

You can imagine now having multiple git repos that you want to share as team specific utilities or bootstrapping. All someone has to do is check out your folder and add it to their environment.

## Features

Things EnvZ gives you

- Opinionated bootstrap (installs python, pip, checks for yadr, gives you a default gui editor [atom])
- Simplifies reloading your .zsh environment and autocompletes
- Lets you break up your environment into multiple folders or git repos and link them to yours with simple commands with autocomplete
- Defines a clean place to add zsh completion files
- Auto sets up github enterprise to work with `hub` and provides json command line parsing with `jq`
- Auto installs cask if its not there
- A hook into your teammates source directory. If everyone uses EnvZ then you can be assured that $SRC\_DIR is set
- Easier pull request autocomplete (via the `pr` command, the current hub one is broken) that lets you pick the target branch and a message

I like yadr and other opinionated setups because they get you up and running fast, but I always found that once you are up and running those bootstrappers didn't thin about how to share your configurations and other utilities. With a team of 3 or 4 people you may want to have some scripts for you, some for the team, and maybe some for side projects that you can have people bootload up.

If thats the case, EnvZ is a good option. It's lightweight, easy to change, and easy to load up new envs with.

Enjoy!

