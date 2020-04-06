---
layout: post
title: IxD 2013 - Production ready CSS workshop
date: 2013-01-28 11:22:49.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- conference
- css
- Design
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1051567750'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560912354;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3500;}i:1;a:1:{s:2:"id";i:4631;}i:2;a:1:{s:2:"id";i:4028;}}}}

permalink: "/2013/01/28/ixd-2013-production-ready-css-workshop/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/ixd-2013-production-ready-css-workshop/)_

Carlo and I are in Toronto for [Ixd 2013](http://interaction13.ixda.org/) for the week hoping to pick up some interesting info on interaction and design. We were painfully reminded of the need for continual design improvement within the first hour of stepping into Canada. We rented a Hyundai Veloster and for 30 minutes couldn't figure out how to start the car. For the uninitiated, you have to hold the brake pedal down while pushing the start button.

[![velocsterStarter](http://tech.blinemedical.com/wp-content/uploads/2013/01/velocsterStarter-300x214.jpg)](http://tech.blinemedical.com/wp-content/uploads/2013/01/velocsterStarter.jpg)

Today the conference started and we attended a workshop called "Sitting in the drivers seat: Designing production level CSS". Carlo's been doing html and css for a long time and was interested to hear what the speaker would say, whereas I am less experienced and wanted to see if I could pick up any design tips. We were pleasantly surprised when [Jack Moffett](http://designaday.tumblr.com/) started and stressed that we should be focusing on blurring the lines between designer and developer. He quoted[Jared Spool](http://www.uie.com/about/consultants/) who said that designers should stop asking "_do I need to code_" but instead should ask "_will learning code help me be a better designer?_". Moffett stressed using developer oriented tools such as version control--svn or git--wikis to track requirements, diff tools, bug tracking software, some sort of IDE for live editing of html and css, and of course the standard Firebug, Chrome, and even Internet Explorer developer toolsets. Using production tools that developers also use integrates the application design and development processes. With wiki's and version control you get historical visibility about changes and decisions. This helps designers know when developers want changes, and helps developers know when designers want changes. Conversations and issues can be tracked in bug trackers like JIRA or FogBugz and gives everyone a sense of direction.

Moffett said that a benefit to following developer practices and learning at least some basic html and css structures is that designers can design with a practical implementation in mind. Understanding the limitations of html and css also helps the designer avoid implementation pitfalls and minimizes the amount of design, implement, tweak rounds during application development. Just like an architect should know the structure and the needs of the building crew, if the design is impracticable it doesn't matter how pretty it is.

After the tool chain introduction Moffett led everyone through [several examples](https://github.com/jackmoffett/DriverSeat) of creating reusable CSS stressing seperating content from skin. The content should be able to stay independent, while skin elements (such as font, padding, etc) can be toggled with css. The example that I think the workshop most enjoyed was taking this:

[![amazonRow.](http://tech.blinemedical.com/wp-content/uploads/2013/01/amazonRow.-300x97.png)](http://tech.blinemedical.com/wp-content/uploads/2013/01/amazonRow..png)

and refactoring it to this:

[![amazonCol.](http://tech.blinemedical.com/wp-content/uploads/2013/01/amazonCol.-167x300.png)](http://tech.blinemedical.com/wp-content/uploads/2013/01/amazonCol..png)

By removing inline css and a few minor content structure tweaks. Now we can toggle between row and column layout using the same html, just with different css. I was a little surprised that Amazon's site was used for the example - I expected at first that Moffett would show it as an example of what to do, not what to NOT do.

Moffett also worked through an example leveraging CSS inheritance to control visible state. By doing it this way you can avoid direct css style manipulations in javascript and use css classes to define visual behaviors. For example, assume we want to control visiblity of a div that contains some photos lets set up the html and css like so.

Photos aren't visible:

[html]  
\<div\>  
 \<div class="photos"\>  
 // some photos  
 \</div\>  
\</div\>  
[/html]

Photos are visible:

[html]  
\<div class="showPhotos"\>  
 \<div class="photos"\>  
 // some photos  
 \</div\>  
\</div\>  
[/html]

The css for this example would look like this:

[css]  
.photos {  
 display:none;  
}

.showPhotos .photos{  
 display:block;  
}  
[/css]

By default the photos div isn't visible, however if we added the `showPhotos` class to the parent, the parents display property will override the `photos` display setting it to `block`. By removing the `showPhotos` class we can hide the element since the photos display element goes back to the default. This way you can control visibility and state by toggling items at the parent level instead of the child level. Carlo also brought up in the seminar that by doing it this way you are setting it up for using css transitions and animations since you can now control state information just with css.

While it may sound trivial to UI developers or designers, it's always good to ground yourself in how to make something simple and reusable. Even the best developers need reminders.

