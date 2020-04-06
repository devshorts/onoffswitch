---
layout: post
title: Installing leinigen on windows
date: 2015-03-14 22:20:02.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- clojure
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560799432;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:8009;}i:1;a:1:{s:2:"id";i:4327;}i:2;a:1:{s:2:"id";i:1268;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/03/14/installing-leinigen-windows/"
---
Figured I'd spend part of the afternoon and play with clojure but was immediately thwarted trying to install [leiningen](http://leiningen.org/) on windows via powershell. I tried the msi installer but it didn't seem to do anything, so I went to my `~/.lein/bin` folder and ran

[code]  
.lein\bin\> .\lein.bat self-install  
Downloading Leiningen now...  
SYSTEM\_WGETRC = c:/progra~1/wget/etc/wgetrc  
syswgetrc = C:\Program Files (x86)\Gow/etc/wgetrc  
--2015-03-14 15:08:48-- https://github.com/technomancy/leiningen/releases/download/2.5.1/leiningen-2.5.1-standalone.zip  
Resolving github.com... 192.30.252.131  
Connecting to github.com|192.30.252.131|:443... connected.  
ERROR: cannot verify github.com's certificate, issued by `/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA':  
 Unable to locally verify the issuer's authority.  
To connect to github.com insecurely, use `--no-check-certificate'.  
Unable to establish SSL connection.

Failed to download https://github.com/technomancy/leiningen/releases/download/2.5.1/leiningen-2.5.1-standalone.zip  
[/code]

Hmm, thats weird. For some reason the cert isn't validating with wget (that I have installed via Gow).

A quick google showed that this is a common problem using the gow wget, and I wasn't about to use the unsecured certificate check. I opened up the leinigen installer bat file and saw that it does a check trying to see what kind of download function your shell has. It checks if you have wget, curl, or if you are in powershell (in which case it creates a .net webclient and downloads the target file).

Since I have gow in my path wget comes up first, so I just switched around the order and things now work happy!

The relevant section in the lein.bat file is

[code]  
:DownloadFile  
rem parameters: TargetFileName Address  
if NOT "x%HTTP\_CLIENT%" == "x" (  
 %HTTP\_CLIENT% %1 %2  
 goto EOF  
)  
call powershell -? \>nul 2\>&1  
if NOT ERRORLEVEL 1 (  
 powershell -Command "& {param($a,$f) (new-object System.Net.WebClient).DownloadFile($a, $f)}" ""%2"" ""%1""  
 goto EOF  
)  
call curl --help \>nul 2\>&1  
if NOT ERRORLEVEL 1 (  
 rem We set CURL\_PROXY to a space character below to pose as a no-op argument  
 set CURL\_PROXY=  
 if NOT "x%HTTPS\_PROXY%" == "x" set CURL\_PROXY="-x %HTTPS\_PROXY%"  
 call curl %CURL\_PROXY% -f -L -o %1 %2  
 goto EOF  
)  
call Wget --help \>nul 2\>&1  
if NOT ERRORLEVEL 1 (  
 call wget -O %1 %2  
 goto EOF  
)  
[/code]

Once the self install completes now lein is available.

On a side note, I think you probably could have just downloaded the release file and plopped it into the `~/.lein/self-installs` folder and it would work too

