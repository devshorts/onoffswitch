---
layout: post
title: Getting battery percentage in zsh
date: 2015-03-30 01:15:14.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- osx
- shell
- zsh
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560766546;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4631;}i:1;a:1:{s:2:"id";i:4589;}i:2;a:1:{s:2:"id";i:4463;}}}}

permalink: "/2015/03/30/battery-percentage-zsh-osx-maverick/"
---
I'm on osx maverick still at home on my laptop and I spent part of today dicking around customizing my zsh shell. I wanted to be able to show my battery percentage in the shell and it's really pretty easy.

First, the main shell function

[bash]  
function get\_battery()  
{  
 current\_battery=`system_profiler SPPowerDataType | grep -i "charge remaining" | awk '{print $4}'`

max\_battery=`system_profiler SPPowerDataType | grep -i "full charge capacity" | awk '{print $5}'`

percent=`bc \<\<\< "scale=4; ${current_battery}/${max_battery} * 100"`

printf '%.0f' $percent  
}  
[/bash]

This queries from the profiler the charge current and remaining, uses bc to get a floating point division, and then just shows the integer value of that (we could round it too but I was lazy).

Now, we just need to tie it into the prompt. I'm using the [steef](https://github.com/robbyrussell/oh-my-zsh/blob/master/themes/steeef.zsh-theme) prompt by default and just tweaked it a bit:

[bash]  
battery\_percentage="$(get\_battery)%%"

directory\_info="${\_prompt\_steeef\_colors[5]}%~%f"

datetime="%F{yellow}[%D\{\%a %I:%M:%S %p}]%f" # yellow [Day hh:mm:ss am/pm] followed by reset color %f

PROMPT="${battery\_percentage} ${datetime} ${directory\_info} ${vcs}  
""$ "

RPROMPT=""  
[/bash]

The two percent symbols just escapes the percent symbol.

Make sure the `battery_percentage` function is defined before you load your prompt in your .zshrc file.

Now here's what I got:

![Screen Shot 2015-03-29 at 6.13.50 PM](http://onoffswitch.net/wp-content/uploads/2015/03/Screen-Shot-2015-03-29-at-6.13.50-PM.png)

