---
layout: post
title: Leveraging message passing to do currying in ruby
date: 2014-08-24 02:43:58.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories: []
tags:
- lambdas
- ruby
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560523888;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4306;}i:1;a:1:{s:2:"id";i:2020;}i:2;a:1:{s:2:"id";i:3565;}}}}

permalink: "/2014/08/24/leveraging-message-passing-currying-ruby/"
---
I'm not much of a ruby guy, but I had the inkling to play with it this weekend. The first thing I do when I'm in a new language is try to map constructs that I'm familiar with, from basic stuff like object instantiation, singletons, inheritance, to more complicated paradigms like lambdas and currying.

I came across this [blog post](http://www.sitepoint.com/functional-programming-techniques-with-ruby-part-ii/) that shows that ruby has a way to auto curry lambdas, which is actually pretty awesome. However, I was a little confused by the syntax

[ruby]  
a.send(fn, b)  
[/ruby]

I'm more used to ML style where you would do

[code]  
fn a b  
[/code]

So what is `a.send` doing?

## Message passing

Ruby exposes its dynamic dispatch as a message passing mechanism (like objective c), so you can send "messages" to objects. It's like being able to say "hey, execute this function (represented by a string) on this context".

If you think of it that way, then `a.send(fn, b)` translates to "execute function 'fn' on the context a, with the argument of b". This means that fn better exist on the context of 'a'.

As an example, this curries the multiplication function:

[ruby]  
apply\_onContext = lambda do |fn, a, b|  
 a.send(fn, b)  
end

mult = apply\_onContext.curry.(:\*, 5)

puts mult.(2)  
[/ruby]

This prints out `10`. First a lambda is created that sends a message to the object 'a' asking it to execute the the function `*` (represented as an [interned string](http://en.wikipedia.org/wiki/String_interning)).

Then we can leverage the curry function to auto curry the lambda for us creating almost F# style curried functions. The syntax of ".(" is a shorthand of .call syntax which executes a lambda.

If we understand message passing we can construct other lambdas now too:

[ruby]  
class Test  
 def add(x, y)  
 x + y  
 end

def addOne  
 apply\_onClass = lambda do |fn, x, y|  
 send(fn, x, y)  
 end

apply\_onClass.curry.(:add, 1)  
 end  
end

puts Test.new.addOne.(4)  
[/ruby]

This returns a curried lambda that invokes a message :add on the source object.

## Getting rid of the dot

Ruby 1.9 doesn't let you define what `()` does so you are forced to call lambdas with the dot syntax. However, ruby has other interesting features that let you alias a method to another name. It's like moving the original method to a new name.

You can do this to any method you have access to so you can get [the benefits](http://www.leonardoborges.com/writings/2008/08/07/why-i-like-ruby-1-alias_method/) of method overriding without needing to actually do inheritance.

Taking advantage of this you can actually hook into the default missing message exception on object (which is invoked when a "message" isn't caught). Catching the missing method exception and then executing a .call on the object (if it accepts that message) lets us fake the parenthesis.

Here is a [blog post](https://github.com/coderrr/parenthesis_hacks/blob/master/lib/lambda.rb) that shows how to do it.

Obviously it sucks to leverage exception handling, but hey, still neat.

## Conclusion

While nowhere near as succinct as f#

[fsharp]  
let addOne = (+) 1  
[/fsharp]

But learning new things about other languages is interesting :)

