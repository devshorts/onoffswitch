---
layout: post
title: 'Tech Talk: Text Editors'
date: 2013-04-18 15:55:59.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- markdown
- text editors
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559335486;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4568;}i:1;a:1:{s:2:"id";i:4244;}i:2;a:1:{s:2:"id";i:3536;}}}}

permalink: "/2013/04/18/tech-talk-text-editors/"
---
Today's tech talk was a little less tech but no less important. We got together and talked about the different text editors that we use and why we like them.

## JuJu Edit

- Pros: We use JuJu at work a lot because it handles enormous files really well. And when I mean enormous I mean upwards of 50+GB log files. Other editors choke when its this big but JuJu handles it fine. On top of that JuJu lets you do custom syntax highlighting, which is great for log files, since now you can highlight Debug, Warn, Info, Errors, etc. It's pretty bare bones but its very lightweight and useful for quick note taking and basic text editing. Also having the ability to auto reload the file on changes makes it great as a `tail` replacement.
- Cons: JuJu isn't in active development anymore and the documentation is non-existant (setting up the regex to do log line highlighting was a lot of trial and error). If you set up your regex for highghlitng wrong it can really choke JuJu and make it run super slow. Also, with large files sometimes it can grind to a halt if you need to scroll to an arbitrary position in an enormous file.
- [http://jujusoft.com/jujuedit/](http://jujusoft.com/jujuedit/)

## EditPad Lite

- Pros: The team commented that EditPad Lite handles large files pretty well, has undo/redo support, auto-save, and a multi tab interface to handle lots of open files. 
- Cons: No syntax highlighting and the multi tab interface. 
- [http://www.editpadlite.com/](http://www.editpadlite.com/)

## EditPad Pro

- Pros: On top of what you get with EditPad you also get syntax highlighting which is cool.
- Cons: Not much other than its not free
- [http://www.editpadpro.com/](http://www.editpadpro.com/)

## Notepad++

- Pros: Notepad++ is a pretty big winner in the text editor shootout. It has auto-reload on change, plugin support, great find in file support, it's actively maintained. It also lets you do cool block/column selection (great for selecting code without the leading indentations).
- Cons: A common complaint was that it's a heavier application than something like JuJu edit. Even with medium sized files (a few hundred MB) the memory usage of the application can soar. So while this may make a great actual text editor, it makes for a poor log viewer.
- [http://notepad-plus-plus.org/](http://notepad-plus-plus.org/)

## Intype

- Pros: Intype is a new player and it's being actively developed. It has syntax highlighting and commands related to specific file types. Also, it lets you handle language fallback fonts, so if you are dealing with different languages this kind of cool. If the default font doesn't have the character set that a language wants to load then it can fallback to another font, and then another, and another. Great if you are working with localization.
- Cons: None yet!
- [http://inotai.com/intype/](http://inotai.com/intype/)

## Markdown Pad

- Pros: I mentioned this at our talk because I like to take notes in markdown. Markdown Pad also lets you put in github flavorerd markdown for code highlighting, and you can render the final markdown as html (with inline CSS), or as a PDF. You write markdown in a left panel and it renders the markdown in a right panel, so you can get a preview of what you are writing as you write it. It has a bunch of other really nice features that I like and for note taking I think its better than just a plain text editor.
- Cons: To get github flavored markdown you need to pay, also to get a bunch of other neat features (like PDF support) you need to get the pro version. THankfully its reasonably cheap and I've gone ahead and done it. One thing I'm not too fond of is the delay in rendering markdown. I think this is because they are doing some optimization to handle very large files, but it would be nice if they did it only when a file reached a certain point and not all the time.
- [http://markdownpad.com/](http://markdownpad.com/)

For those who are wondering, we did discuss vim, emacs, textmate, and scintilla, but didn't spend much time talking about them hence why I didn't include it. I've personally used vim a lot in my linux days but it's not my personal choice.

Text editors are a highly personal choice, so it's great that there are so many tools out there. The paid versions of these apps are usually relatively cheap and if you like a piece of software definitely buy it!

