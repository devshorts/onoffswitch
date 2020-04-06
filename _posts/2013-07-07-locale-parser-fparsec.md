---
layout: post
title: Locale parser with fparsec
date: 2013-07-07 08:00:22.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- fparsec
- fsharp
- language implementation
- parser
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1554392642;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3723;}i:1;a:1:{s:2:"id";i:4131;}i:2;a:1:{s:2:"id";i:4077;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/07/07/locale-parser-fparsec/"
---
Localizing an application consists of extracting out user directed text and managing it outside of hardcoded strings in your code. This lets you tweak strings without having to recompile, and if done properly, allows you to support multiple languages. Localizing is no easy task, it messes up spacing, formatting, name/date other cultural information, but thats a separate issue. The crux of localizing is text.

But, who just uses bare text to display things to the user? Usually you want to have text be a little dynamic. Something like

[code]  
Hello {user}! Welcome!  
[/code]

Here, `user` will be some sort of dynamic property. To support this, your locale files need a way to handle arguments.

One way of storing contents in a locale file is like this:

[code]  
ExampleText = Some Text {argName:argType} other text etc  
 = This is on a seperate newline  
UserLoginText = ...  
[/code]

This consists of an identifier, followed by an equals sign, followed by some text with arguments of the form {x:y}. To make a new line you have a new line with an equals sign and you continue your text. When you reach a string with an identifier, you have a new locale element.

But you can also have comments, like

[code]  
# this is a comment, ignore me!  
[/code]

And to throw a monkey wrench in the problem, you can also have arguments with no types, of the form `{argName}`.

The end goal, is to be able to reference your locale contents in code, something like

[code]  
Locale.ExampleText ("foo");  
[/code]

But to get to the point where you can reference this you need to translate your locale files into something workable, kind of like a syntax tree. If you have a working syntax tree of your locale files you can generate strongly typed locale code for you to use in your application.

## The data

To parse a locale file of this format I used fparsec. One reason was that it already handles lookaheads and backtracking, and another reason is that I wanted to play with it :)

Going with a data first design, I thought about what I wanted to my final output to be and came up with 3 discriminated unions that look like this:

[fsharp]  
type Arg =  
 | WithType of string \* string  
 | NoType of string

type LocaleContents =  
 | Argument of Arg  
 | Text of string  
 | Line of LocaleContents list  
 | Comment of string  
 | NewLine

type Locale =  
 | Entry of string \* LocaleContents  
 | IgnoreEntry of LocaleContents  
[/fsharp]

## Utilities

The next step was to build out some common utilities that I can use. I knew I'd need to be able to parse a phrase, a single word, and know when things are between brackets:

[fsharp]  
(\*  
 Utilities  
\*)

let brackets = isAnyOf ['{';'}']

(\* non new line space \*)  
let regSpace = manySatisfy (isAnyOf [' ';'\t'])

(\* any string literal that is charcaters \*)  
let phrase = many1Chars (satisfy (isNoneOf ['{';'\n']))

let singleWord = many1Chars (satisfy isDigit \<|\> satisfy isLetter \<|\> satisfy (isAnyOf ['\_';'-']))

(\* utility method to set between parsers space agnostic \*)  
let between x y p = pstring x \>\>. regSpace \>\>. p .\>\> regSpace .\>\> pstring y  
[/fsharp]

Fparsec comes with a lot of great functions and parser combinators to create robust parsers. The idea is to combine parser functions from smaller parsers into larger parsers. I liked working with it because it felt like dealing directly with a grammar.

## Arguments

Now that I was able to parse words, phrases, and I could seperate out newlines from spaces, lets tackle an argument:

[fsharp]  
(\*  
 Arguments of {a:b} or {a}  
\*)

let argDelim = pstring ":"

let argumentNoType = singleWord |\>\> NoType

let argumentWithType = singleWord .\>\>.? (argDelim \>\>. singleWord) |\>\> WithType

let arg = (argumentWithType \<|\> argumentNoType) |\> between "{" "}" |\>\> Argument  
[/fsharp]

The `.>>.?` combinator says to apply both combinators results as a tuple, but if it fails to backtrack to the state of the previous parser. Also, the `` combinator lets you apply parsers as alternatives, so either of the parsers can be applied.

## Text elements

Next up is text elements. This is the contents after the = of the identifier, but not including arguments. For example, if our locale entry is

[code]  
UserLogin = Hey! Whats up?  
 = new lineezzz  
[/code]

We want to match on "Hey! Whats up?", followed by an explict newline, followed by "new lineeezz"

[fsharp]  
(\*  
 Text Elements  
\*)

let textElement = phrase |\>\> Text

let newLine = (unicodeNewline \>\>? regSpace \>\>? pstring "=") \>\>% NewLine

let line = many (regSpace \>\>? (arg \<|\> textElement \<|\> newLine)) |\>\> Line  
[/fsharp]

Remembering that a phrase is any text except for a start bracket and a newline, we can parse all text up to an argument. New lines are a new line, followed by some space (maybe), followed by an equal sign. Since the newline doesn't contain any data we care about from the parser we can ignore the output and just assign the result to the union type `NewLine` using the `>>%` operator.

But a line is an aggregation of new lines, arguments, and phrases, so we can use the fparsec `many` operator, along with the 3 alternatives (arguments, text elements, and new lines) to build out an actual line.

## An Entry

Since we have arguments, new lines, and text set up, we can finally put it all together. What I need now is to match when we have an identifier ("UserLogin"), an equals sign, followed by a line.

[fsharp]  
(\*  
 Entries  
\*)

let delim = regSpace \>\>. pstring "=" .\>\> regSpace

let identifier = regSpace \>\>. singleWord .\>\> delim

let localeElement = unicodeSpaces \>\>? (identifier .\>\>. line .\>\> skipRestOfLine true) |\>\> Entry  
[/fsharp]

This gives you a tuple of identifier \* line, representing your entire locale element.

## Comments

But we also have to account for comments. Thankfully those are pretty easy

[fsharp]  
(\*  
 Comments  
\*)

let comment = pstring "#" \>\>. restOfLine false |\>\> Comment

let commentElement = unicodeSpaces \>\>? comment |\>\> IgnoreEntry  
[/fsharp]

This says if you match a "#" then take the rest of the line (but leave the newline since other parsers will handle that). We might as well maintain the comment information so we can pipe that result to the `IgnoreEntry` union type.

## Running the parser

And now we just have to piece together comments, locale elements, and run the parser

[fsharp]  
(\*  
 Full Locale  
\*)

let locale = many (commentElement \<|\> localeElement) .\>\> eof

let test input = match run locale input with  
 | Success(r,\_,\_) -\> r  
 | Failure(r,\_,\_) -\>  
 Console.WriteLine r  
 raise (Error(r))  
[/fsharp]

## Example

Lets try it out. Here is my sample locale:

[code]  
UserLogin = {user}! Whats up!  
 = You rock, thanks for logging

UserLogout = {firstName:string}, {lastName:string}...why you gotta go? We were just getting to know you  
[/code]

And running it in fsi

[fsharp]  
\> test "UserLogin = {user}! Whats up!  
 = You rock, thanks for logging

UserLogout = {firstName:string}, {lastName:string}...why you gotta go? We were just getting to know you ";;

val it : Locale list =  
 [Entry ("UserLogin", Line [Argument (NoType "user"); Text "! Whats up!"; NewLine; Text "You rock, thanks for logging"]);  
 Entry ("UserLogout", Line  
 [Argument (WithType ("firstName","string")); Text ", ";  
 Argument (WithType ("lastName","string"));  
 Text "...why you gotta go? We were just getting to know you "])]  
[/fsharp]

And now its easy to iterate and manipulate the data!

## Source

Full source available at [my github](https://github.com/devshorts/Locale-parser/blob/master/FParsecCombinators/Program.fs)

