---
layout: post
title: 'Tech Talk: AngularJS'
date: 2013-05-02 16:40:05.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tech talks
tags:
- angularjs
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558720449;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4028;}i:1;a:1:{s:2:"id";i:4515;}i:2;a:1:{s:2:"id";i:3500;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/05/02/tech-talk-angularjs/"
---
Today's tech talk was a continuation on front-end discussions we're having. Last week we talked about typescript (I forgot to write it up) and this week we discussed the basics of [angular](http://angularjs.org/). Angular is a front-end MVC framework written by google that, at first glance, looks completely different from previous javascript/html development. The basic gist is to strongly decouple logic into encapsulated modules. But that's not all there is, there's a lot to it. Angular has a templating engine, dependency injection, double bindings between views and controllers, event dispatching, etc.

Since it's hopeless to cover all of angular in one blog post I'll just mention a few of the good questions that came up during our discussion

- _Can a child component dispatch to a parent component? Let's say something happened in an innner component and you need a parent compoinent to also register that?_  
Sure, angular provides a function called `emit` that lets a child component redispatch an event to parent components, and a `broadcast` method for a parent component to dispatch to all child components.
- _Why use client side rendering vs server side rendering?_  
This was an interesting discussion we had. While we didn't come up with a definite answer, we figure it's partial because browsers can handle more work now, and partially beacuse the trend is to decouple the server from view logic. Servers are more often being designed as a collection of REST apis and to serve static documents, so the UI can now handle compilation of a view and offload that work onto a client. So why does server side rendering still exist? Well, it's the way its always been done and it's easy to do. On top of that it does offload client side work. I guess it's two schools of thought of when to do what.
- _How do you profile angular?_  
This was an interesting discussion. We didn't know of any particular specific ways to profile angular other than built in profiling tools in browsers and other external html/js profilers, but angular has done a really nice job of exposing almost the entire framework in a [unit testable way](http://docs.angularjs.org/guide/dev_guide.unit-testing). This means you can step through angular and see where bottlenecks are within the framework.
- _What if angular stops being developed, can you use a later version of JQuery if their bundled version of JQuery lite stops being updated?_  
Another good question. You can alias new version of JQuery (apparenlty JQuery already has this feature), but you can also provide a new JQuery object as a provider to any directives if you wanted to. This way you can have the original JQuery that angular comes with, and you can have the injected secondary JQuery that you can use.
- _Is it possible to mix server side rendering (with Razor or Jade) with angular client side templating?_  
Sure, since whatever the server doesn't recognize (in the {{ ... }} syntax) will be rendered in the client side.
- _Does angular come with prepackged widgets?_  
Not really, but it's easy to wrap any other frameworks/libraries widgets within directives.

## More Info

Here are some links that talk more about angular

Video Tutorial: AngularJS Fundamentals in 60-ish Minutes: [http://weblogs.asp.net/dwahlin/archive/2013/04/12/video-tutorial-angularjs-fundamentals-in-60-ish-minutes.aspx](http://weblogs.asp.net/dwahlin/archive/2013/04/12/video-tutorial-angularjs-fundamentals-in-60-ish-minutes.aspx)

Angular Providers: [http://slides.wesalvaro.com/20121113/#/](http://slides.wesalvaro.com/20121113/#/)

AngularJS MTV Meetup: Best Practices (2012/12/11): [http://www.youtube.com/watch?v=ZhfUv0spHCY](http://www.youtube.com/watch?v=ZhfUv0spHCY)

Excellent set of short video tutorials on AngularJS (recommended!): [http://www.egghead.io/](http://www.egghead.io/)

[http://www.egghead.io/video/HvTZbQ\_hUZY](http://www.egghead.io/video/HvTZbQ_hUZY)

Code Organization in Large AngularJS and JavaScript Applications: [http://cliffmeyers.com/blog/2013/4/21/code-organization-angularjs-javascript  
](http://cliffmeyers.com/blog/2013/4/21/code-organization-angularjs-javascript)

Building up AngularJS (Greg Weber yap.TV): [http://www.slideshare.net/antonkropp/angular-js-meetup-20416779](http://www.slideshare.net/antonkropp/angular-js-meetup-20416779)  
[http://stephanebegaudeau.tumblr.com/post/48776908163/everything-you-need-to-understand-to-start-with](http://stephanebegaudeau.tumblr.com/post/48776908163/everything-you-need-to-understand-to-start-with)

