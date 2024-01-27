---
layout: post
title: Instagram viewer with node and angular
date: 2014-04-28 08:00:51.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- angular
- instagram
- node
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560156459;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1587;}i:1;a:1:{s:2:"id";i:4783;}i:2;a:1:{s:2:"id";i:4699;}}}}

permalink: "/2014/04/28/instagram-viewer-node-angular/"
---
I have an artist [buddy](http://www.jeremyjams.com/) who is working on an art installation and asked me if there was a way to display a realtime view of an instagram hashtag feed on a projector.

Unfortunately there isn't anything right out of the box available, but I offered to write him a quick app that he could fire up that would give him what he wanted.

_For the impatient, full source is available at my [github](https://github.com/devshorts/Playground/tree/master/Gecker)._

One way to do this would be to hook into the instagram [realtime API](http://instagram.com/developer/realtime/). Using the realtime API you subscribe to tags or users via their API and supply a url callback. Instagram will then callback to your endpoint with an HTTP GET, validate that you actually did request (via a handshake response), and then post to your endpoint whenever something on that subscription has changed. What it won't do is actually give you the information you need to display, it's only a notification that something changed (and gives you information to pull back what changed).

On the one hand this would work great, but the downside is that it requires a publicly exposed endpoint to work. Given that I'm distributing this to a buddy who may not have access to the router at the art installation to set up port forwarding, I went with a more low-tech solution: rss tag polling.

My first idea was to just have this be a UI only page that pulled from the instagram RSS by tag feed periodically, but I ran into cross site origin failures. In order to get over that I wrote a small node app that does the rss tag query on a timer, xml parsing, xml data translation (into a more useful DTO), and uses socket.io to push the new information to a simple angular app.

## The server

First I abstracted the concept of a server into its own class where we could inject callbacks if we wanted to externally. I exposed the routing as an injection function so any other consumers can add routes to the expression app (so if I wanted to build the realtime API I could) without the server caring at all what's going on.

```js
  
var express = require('express');  
var http = require('http');  
var app = express();  
exports.Server = function(){  
 var hostRoot = \_\_dirname + '/../ui';

console.log(hostRoot);

app.use(express.bodyParser());  
 app.use(express.methodOverride());  
 app.use(app.router);  
 app.use(express.static(hostRoot));  
 app.use(express.errorHandler());

this.start = function(){  
 var server = http.createServer(app);

var port = process.env.PORT || 3000;  
 server.listen(port);

console.log("listneing on " + port);

return server;  
 };

this.addRoutes = function(callback){  
 callback(app);  
 };  
};  

```

I also added a class that abstracts socket.IO and lets us issue an action on connect, as well as external invoking an action. The idea here is that the moment someone connects we want to make sure to send the most to date data. Given that the client load will realistically be only 1, maybe 2 people, there isn't an issue of server spam here.

```js
  
var io = require('socket.io');

exports.RealTime = function(server){

var socketIO = io.listen(server);

socketIO.set('log level', 1);

var root = this;

this.onLogin = function(pushTo){  
 root.loginFunction = pushTo;

return root;  
 };

this.run = function(){  
 socketIO.sockets.on('connection', function(socket){  
 console.log("connected");

socket.on("disconnect", function(){  
 console.log("disconnect");  
 });

root.loginFunction(root.push);  
 });

return this;  
 };

this.push = function(data) {  
 socketIO.sockets.json.emit("data", data);  
 }  
};  

```

I also have a class that encapsulates pulling data from the instagram RSS feed by tag and transforms the result into a simpler object

```js
  
var request = require('request');  
var \_ = require('underscore').\_;  
var xml2js = require("xml2js");  
var Instagram = require("instagram-node-lib");

exports.InstagramRss = function(tag, takeAmount){  
 var options = {  
 host: "http://instagram.com/tags/" + tag + "/feed/recent.rss",  
 method: 'GET'  
 };

this.query = function(callback){  
 function extractor(body){  
 var parser = xml2js.Parser();

return parser.parseString(body, function(err, r){  
 var items =  
 \_.chain(r.rss.channel[0].item)  
 .map(function(element){  
 return {  
 link : element.link[0],  
 title: element.title[0]  
 }  
 })  
 .take(takeAmount)  
 .value();

callback(items);  
 });  
 }

request(options.host, function (error, response, body) {  
 if (!error && response.statusCode == 200) {  
 extractor(body);  
 }  
 })  
 };  
};  

```

Now finally the node entrypoint

```js
  
var openurl = require("openurl");

var Server = require("./src/server").Server,  
 RealTime = require("./src/realtime").RealTime,  
 InstagramRss = require("./src/instagramRss").InstagramRss;

var App = function(){

var config = require('./config.json');

var rss = new InstagramRss(config.tag, config.take);

var server = new Server();

var realtime = {};

this.run = function(){  
 runOnTimer(config.interval);

realtime = new RealTime(server.start()).onLogin(rss.query).run();  
 };

function runOnTimer(interval){  
 setInterval(function(){  
 rss.query(realtime.push)  
 }, interval \* 1000);  
 }  
};

new App().run();

openurl.open('http://localhost:3000');  

```

The idea now is that anyone who connects to the websocket will immediately get an rss query pushed to them. From then on at an interval configured by config.json we'll just send any new stuff to them.

## The UI

The ui is dirt simple. It's just a single angularJS page that registers a service and callback representing the realtime push, as well as a controller and a directive to manage displaying new instagram elements.

The main angular app, below, takes care of registering services and directives, as well as the initial routing (using ui-router)

```js
  
function App(){  
 this.run = function(app){  
 new ServiceInitializer().initServices(app);  
 new Directives().initDirectives(app);

applyConfigs(app);  
 };

function applyConfigs(app){  
 app.config(function($stateProvider, $urlRouterProvider){

$urlRouterProvider.otherwise("/");

$stateProvider.state('main', {  
 url:"/",  
 templateUrl: "partials/feed.html",  
 controller: feedController  
 })  
 });  
 }  
}  

```

The service initializer and services are just wrappers on the realtime subscription

```js
  
function ServiceInitializer(){  
 this.initServices = function (app){  
 app.service('realtime', realtime);  
 };

function realtime(){  
 function basePath(){  
 var pathArray = window.location.href.split( '/' );  
 var protocol = pathArray[0];  
 var host = pathArray[2];  
 return protocol + '//' + host;  
 }

var socket = io.connect(basePath());

var rssQueryClients = [];

socket.on('data', function(data){  
 \_.forEach(rssQueryClients, function(client){  
 client(data);  
 });  
 });

this.registerRssPush = function (client){  
 rssQueryClients.push(client);  
 };  
 }  
}  

```

And the directive that drives the single image display. I set it up so that the image and tagged text fade in together when the image has completed loading.

```js
  
function Directives(){  
 this.initDirectives = function(app){  
 app.directive('instagram', instagram)  
 };

function instagram(){  
 return {  
 restrict: 'E',  
 scope: {  
 data:"="  
 },  
 templateUrl: 'partials/directives/instagram-directive.html',  
 link: function (scope, element, attrs){  
 var img = $(element).find("img")[0];  
 var txt = $(element).find(".image-text")[0];

$(img).bind("load", function(event){  
 $(img).css("opacity", 1);  
 $(txt).css("opacity", 1);  
 });  
 }  
 };  
 }  
}  

```

The only controller we need is to handle the realtime socket and filtering of the new input data

```js
  
function feedController($scope, realtime, $http){  
 $scope.feed = [];

realtime.registerRssPush(function (data) {  
 console.log("got data");

var map = {};

\_.forEach($scope.feed, function (item){  
 map[item.link] = true;  
 });

\_.forEach(data, function (item) {  
 if(!map.hasOwnProperty(item.link)){  
 $scope.feed.unshift(item);  
 }  
 });

$scope.$apply();  
 });  
}  

```

Since the data comes in via a websocket and not in the scope of a wrapped angular service we need to do a $scope.apply to make sure the page redraws.

The view directive represents one single instagram image

```html
  
\<img src="{{data.link}}"/\>

\<div class="image-text"\>  
 {{ data.title }}  
\</div\>  

```

And the main view is just a repeater of the directives (controlled by the main controller)

```html
  
\<div class="repeat-body"\>  
 \<div class="instagram-element" ng-repeat="item in feed"\>  
 \<instagram data="item"\>\</instagram\>  
 \</div\>  
\</div\>  

```

To drive the whole thing the relevant index page blocks look like

```html
  
///.. js includes etc...

\<script\>  
 var app = angular.module("app", ["ui.router"]);

new App().run(app);  
 \</script\>  
 \</head\>  
 \<body \>  
 \<!-- Add your site or application content here --\>  
 \<div ui-view\>

\</div\>  
 \</body\>  

```

## Conclusion

Apps like this I think node.js really excels at. It's os agnostic, and I can package the whole thing up and send it to my buddy to run. With libraries like underscore js I can even leverage higher order functions and keep things nice and clean.

If I was going to make this a production thing, instead of just a weekend project, I would've used typescript to give myself strong typing.

