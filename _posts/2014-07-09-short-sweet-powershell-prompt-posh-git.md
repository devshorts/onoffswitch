---
layout: post
title: Short and sweet powershell prompt with posh-git
date: 2014-07-09 18:58:33.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- git
- powershell
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561303870;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4892;}i:1;a:1:{s:2:"id";i:4631;}i:2;a:1:{s:2:"id";i:4699;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2014/07/09/short-sweet-powershell-prompt-posh-git/"
---
My company has fully switched to git and it's been great. Most people at work use SourceTree as a gui to manage their git workflow, some use only command line, and I use a mixture of posh-git in powershell with tortoise git when I need to visualize things.

Posh-git, if you load the [example](https://github.com/dahlbyk/posh-git/blob/master/profile.example.ps1) from your profile, will set the default prompt to be the current path. If you go into a git directory it'll also add the git status. Awesome. But if you are frequently in directories that are 10+ levels deep, suddenly your prompt is just obscenely long.

For example, this is pretty useless right?

![2014-07-09 11_53_20-](http://onoffswitch.net/wp-content/uploads/2014/07/2014-07-09-11_53_20-.png)

Obviously it's a fictitious path, but sometimes you run into them, and it'd be nice to optionally shorten that up.

It's easy to define a shortPwd function and expose a global "MAX\_PATH" variable that can be reset.

[code]  
$MAX\_PATH = 5

function ShortPwd  
{  
 $finalPath = $pwd  
 $paths = $finalPath.Path.Split('\')

if($paths.Length -gt $MAX\_PATH){  
 $start = $paths.Length - $MAX\_PATH  
 $finalPath = ".."  
 for($i = $start; $i -le $paths.Length; $i++){  
 $finalPath = $finalPath + "\" + $paths[$i]  
 }  
 }

return $finalPath  
}  
[/code]

In the posh-git example, make sure to load your custom function first, then change

[code]  
Write-Host($pwd.ProviderPath) -nonewline  
[/code]

To

[code]  
Write-Host (ShortPwd) -nonewline -foregroundcolor green  
[/code]

(I like my prompt green)

Now you can dynamically toggle the max length. I've set it to 5, but if you change it the prompt will immediately update:

![2014-07-09 11_57_40-posh~git ~ powershell_scripts [master] (Admin)](http://onoffswitch.net/wp-content/uploads/2014/07/2014-07-09-11_57_40-poshgit-powershell_scripts-master-Admin.png)

For this and other powershell scripts check out my [github](https://github.com/devshorts/powershell_scripts).

