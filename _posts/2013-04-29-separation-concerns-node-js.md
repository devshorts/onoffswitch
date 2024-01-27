---
layout: post
title: Separation of concerns in node.js
date: 2013-04-29 08:00:23.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- node.js
- typescript
meta:
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _edit_last: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560893531;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4515;}i:1;a:1:{s:2:"id";i:4028;}i:2;a:1:{s:2:"id";i:2635;}}}}

permalink: "/2013/04/29/separation-concerns-node-js/"
---
I've been playing with typescript and node.js and I wanted to talk a little about how I've broken up my app source. It's always good to modularize an application into smaller bits, and while node lets you do a lot, quickly, with just a little bit of code, as your application grows you really can't put all your logic in one big `app.ts`.

## App.ts

Instead of the normal application example you see for node projects, I wanted to make it clearer what the initialization of my application does. My app start is structured like this:

[ts]  
/\*\*  
 \* Module dependencies.  
 \*/

import db = module("./storage/storageContainer");  
import auth = module("./auth/oauthDefinitions");  
import requestBase = module("./routes/requestBase");

var express = require('express')  
 , routes = require('./routes')  
 , http = require('http')  
 , path = require('path')  
 , log = require("./utils/log.js")  
 , fs = require('fs')  
 , passport = require('passport');

var app = express();

class AppEntry{  
 constructor(){  
 this.initDb();  
 this.setupRoutes();  
 this.defineOAuth();  
 this.startServer();  
 }

// initialization functions  
}

var application = new AppEntry();  
[/ts]

The upside to this kind of simple structure is that it's easy to see what the entrypoint structure is. Adding new initialization logic is encapsulated and isn't intermingled among route configurations, OAuth authorization code, server start, database initialization, etc. Having a monolithic app can quickly get into a tangled mess.

You may have noticed that I didn't pass in any required modules or references to the application. This is because I'm relying on the class initialization closure to capture the variables to keep function signatures clean. I opted to use a class instead of a module for no particular reason other than I like classes and forgot modules existed when I did this.

## Storage

Even though I'm using mongoose as my mongoDB ORM, I still have tried to move all the storage logic in special storage classes. This means that any outside access to storage has to go through classes that wrap the storage calls. I've mentioned it before, but I think it's always good practice to not entangle an application with specific 3rd party libraries (if you can avoid it). Also having storage classes means I can hide away internal mongo calls, if necessary, and let me do extra data manipulation outside of the context that wants the data.

To make accessing the storage classes easy for myself, I have split them up into separate classes based on what they most commonly access. For example, there is a `userStorage` class, and a `trackStorage` class, etc. Each class contains relevant CRUD and helper methods to aggregate the data in forms that I commonly use them.

Unfortunately, the way node works is that in each module you work in, if you wanted access to a storage class you'd have to import each one independently (one import for users, one import for dataPoints, etc). That's a pain. Instead, I've wrapped the storage classes with a single exported singleton container.

[ts]  
// storageContainer.ts

import schemaImport = module("./schema");  
import users = module("./userStorage");  
import tracks = module("./trackStorage")

export var storage:schemaImport.db = new schemaImport.db();  
export var userStorage:users.userStorage = new users.userStorage();  
export var schema = schemaImport;  
export var trackStorage:tracks.trackStorage = new tracks.trackStorage();  
[/ts]

Anywhere I want access to storage classes, I only need to import one module:

[ts]  
import db = module("./storage/storageContainer");

// ...

db.userStorage.getUserByUsername(...)  
[/ts]

Adding new storage classes and updating the singleton container means I have access to these everywhere I need them without having to worry about importing and instantiating modules.

## Definition files

Like the storage classes, the same pattern goes for definition files. I've made a folder called `def` and created an `all.d.ts` that just has reference path's to all my other definition mappings.

[ts]  
///\<reference path="./mongoose.d.ts"/\>  
///\<reference path="./nodeUnit.d.ts"/\>  
///\<reference path="./schemaDef.d.ts"/\>  
///\<reference path="./passport.d.ts"/\>  
[/ts]

Any other file that needs definition mappings can include the one all aggregate. Since it costs nothing and is just a compiler hint, there's no resource hit.

## Routes

And again, I do the same kind of pattern with routes. I have a folder setup like this:

```
  
routes  
├── index.js  
├── indexRoutes.ts  
├── userRoutes.ts  
├── ... etc  

```

Where index.js looks like this:

[javascript]  
var userRoutes = require("./userRoutes");  
var indexRoutes = require("./indexRoutes");  
var trackRoutes = require("./trackRoutes");  
var partialsRoutes = require("./partialsRoutes");

module.exports = function(app){  
 new userRoutes.userRoutes(app);  
 new indexRoutes.indexRoutes(app);  
 new trackRoutes.trackRoutes(app);  
 new partialsRoutes.partialsRoutes(app);  
};  
[/javascript]

From my main application, I import the routes module and pass it the app reference. I know that [app is global in a node application](http://dailyjs.com/2012/01/26/effective-node-modules/), however, I don't like relying on globals. It was just as easy to pass app as an argument and I prefer that flow control.

[ts]  
routes = require('./routes')  
routes(app);  
[/ts]

In each of the route modules, I then go back to typescript

[ts]  
import db = module("../storage/storageContainer");

import base = module("./requestBase");

export class userRoutes {  
 constructor(app:ExpressApplication) {  
 var requestUtils = new base.requestBase();

app.get('/logout', (req:any, res) =\> {  
 req.logout();  
 res.redirect('/');  
 });

app.get("/users", requestUtils.ensureAuthenticated, (req, res) =\> {  
 res.send(req.user.name);  
 });  
}  
[/ts]

So at this point I'm using the definition files from [DefinitelyTyped](https://github.com/borisyankov/DefinitelyTyped) for my express application. Also you can see that I'm injecting custom middleware into my routes for things that I want to make sure are authenticated. The point being that the routes class is now self encapsulated, and we don't need to modify the main entry point to the application anymore. We update the routes index page and create a route object and that's it.

## Conclusion

It's been fun playing in node, and while I see myself doing some things the .NET way, I'm also trying to embrace the node way. However, when it comes to module organization the node projects I've seen have been seriously lacking. While it does mean more boilerplate upfront, I think making sure to split up your files helps maintain a separation of concerns and your project extensible.

