---
layout: post
title: Event emitters with success and fail methods for node.js
date: 2013-02-27 00:29:03.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- events
- JavaScript
- node.js
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560295314;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3295;}i:1;a:1:{s:2:"id";i:3452;}i:2;a:1:{s:2:"id";i:3128;}}}}

permalink: "/2013/02/27/event-emitters-with-success-and-fail-methods-for-node-js/"
---
When it comes to node.js you hear a lot of hype, good and bad, so I've finally decided to take the plunge and investigate for myself what the fuss is about. So far it's been interesting.

I'm not really building anything in particular right now, I'm just playing with different tech stacks to see how things are done. One of the things that I found I liked, while experimenting with node modules, is the syntax _success_ and _fail_ for callback registration. Something like this:

[javascript]  
module.doSomething()  
 .success(function() { })  
 .fail(function() { });  
[/javascript]

Using this kind of syntax I wanted to have a basic user authentication forwarder that I could wrap route calls in such that only logged in users could call the route. Non logged in users would automatically be forwarded to a twitter oauth route for auto login (done using [everyauth](https://github.com/bnoguchi/everyauth)).

The first step was to create a custom event emitter object. Coming from a .NET world this is similar to creating a class that dispatches an event delegate. To wire up the event handlers you need to register a callback function to execute when the event happens.

For event dispatching to work, your object needs to inherit from the `events` module:

[javascript]  
var EventEmitter = require("events").EventEmitter;  
var sys = require("sys");

sys.inherits(yourObject, EventEmitter);  
[/javascript]

And to emit events, you just emit them by name:

[javascript]  
this.emit('failure');  
[/javascript]

On the consumer end, to listen for events, you register a function on the object's inherited `on` member with the corresponding event name:

[javascript]  
var obj = new yourObject();  
obj.on('someEvent', someFunction)  
[/javascript]

This is great, but I really don't like relying on strings. My biggest qualm with javascript is its lack of strong typing and runtime analysis, so anything I can do to prevent myself from making mistakes is a good thing.

Bringing it back to my original intention, I wanted to encapsulate my authentication checker in a class that could secure API calls. It would validate that the request had a user property and if so asynchronously dispatch success, or if no user was there, a failure. If a failure happened, it would also automatically forward the request to the appropriate authentication route.

Here is what I did

[javascript]  
var EventEmitter = require("events").EventEmitter;  
var sys = require("sys");

function Checker(req, res) {  
 this.req = req;  
 this.res = res;  
 EventEmitter.call(this);  
}

sys.inherits(Checker, EventEmitter);

Checker.prototype.success =  
 function (fct) {  
 this.on('success', fct)  
 return this  
 };

Checker.prototype.failure =  
 function (fct) {  
 this.on('failure', fct)  
 return this  
 };

Checker.prototype.run = function () {  
 if (this.req.user === undefined || this.req.user == null) {  
 this.res.redirect("/auth/twitter");  
 this.emit('failure');  
 }  
 else {  
 this.emit('success');  
 }  
 return this;  
};

module.exports.checkAuth = function (req, res) {  
 return new Checker(req, res);  
};  
[/javascript]

Lets see how it's used:

[javascript]  
app.get("/user/home", function(req, res){  
 utils.checkAuth(req, res)  
 .success(function() {  
 res.send("user is logged in!")  
 })  
 .failure(function(){  
 console.log("requested not logged in")  
 })  
 .run();  
 });  
[/javascript]

After I defined the `Checker` constructor I inherited from the event emitter object. This overrides the prototype of the object, so all other methods should be defined after you do the inheritance. Then I defined a success function that hid the `success` event listener, as well as a `failure` event listener. All the methods return a `this` property to make the method calls [fluent](http://en.wikipedia.org/wiki/Fluent_interface). The `Checker` constructor takes the request and result objects so we can test if the user exists.

To use it, I called the `checkAuth` function on the exported utility class, which returns a new instance of our checker object. Then I register success and failure functions on the returned instance. Lastly, to get things running, I call `run()` which actually tests the code. Now I have a nice reusable authentication forwarder that I can use in any authentication required API calls.

The next step that I was thinking about was trying to see if I can get rid of the `.run()` method. Let's change the constructor wrapper and tester function to look like this

[javascript highlight="15,16,17"]  
Checker.prototype.test = function () {  
 if (this.req.user === undefined || this.req.user == null) {  
 this.res.redirect("/auth/twitter");  
 this.emit('failure');  
 }  
 else {  
 this.emit('success');  
 }  
 return this;  
};

Checker.prototype.run = function(){  
 var self = this;

setTimeout(function(){  
 self.test()  
 }, 1)

return this;  
}

module.exports.checkAuth = function (req, res) {  
 var checker = new Checker(req, res);  
 return checker.run();  
};  
[/javascript]

And calling it like this:

[javascript]  
app.get("/add/:track", function(req, res){  
 utils.checkAuth(req, res)  
 .success(function() {  
 res.send("did it!");  
 })  
 .failure(function(){  
 console.log("requested not logged in")  
 });  
});  
[/javascript]

The main change (highlighted) was to defer the actual testing of the method with a one millisecond timeout. The run method returns the `this` object, which lets you register the success and fail callbacks. One millisecond later the test function fires and assumes the callbacks were registered.

The reason this works is explained in a snippet from John Resig's (creator of JQuery) [blog](http://ejohn.org/blog/how-javascript-timers-work/):

> If a timer is blocked from immediately executing it will be delayed until the next possible point of execution (which will be longer than the desired delay).

By deferring the `test` function we're leveraging node's event loop and the single threaded nature of javascript. Events will process when they get a chance (in the order they were queued), but as long as we are blocking they won't run. So, assuming you register your callbacks during the current execution flow then the timer will be blocked from executing until you are done.

Just to demonstrate that point, here is a modified route. I've intentionally made it a little wonky. We get an `checkAuth` object via a function return, then we add a failure registration. Then we busy wait for a period of time (essentially blocking), then register a success function. Finally we logged that the route function was complete. Then we emit the success event from the deferred timer and log our callback invocation from the success function:

[javascript]  
app.get("/add/:track", function(req, res){

function getAuth(req, res){  
 return utils.checkAuth(req, res);  
 }

var auth = getAuth(req, res);

auth.failure(function(){  
 console.log("requested not logged in")  
 });

var x = 100000000;  
 while(x \> 0){  
 x = x - 1;  
 }

auth.success(function() {  
 console.log("success " + new Date().getTime());  
 res.send('donezo');  
 })

console.log("ready " + new Date().getTime());  
});  
[/javascript]

Notice the while loop that should sit for a few seconds. When you hit `http://localhost:3000/add/test` the console logs

[code]  
registered failure function  
registered success function  
ready 1361995555376  
success 1361995556530  
[/code]

which shows that the timer was deferred until the execution flow returned. When control flow exited, the delayed test function ran, emitting the event and executing our callbacks.

