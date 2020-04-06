---
layout: post
title: Mongoose with TypeScript
date: 2013-03-25 08:00:23.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- JavaScript
- mongoose
- typescript
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561781734;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3295;}i:1;a:1:{s:2:"id";i:4028;}i:2;a:1:{s:2:"id";i:3452;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/25/mongoose-with-typescript/"
---
Mongoose is a library for node.js that wraps the mongoDB driver. Since I've been playing with typescript, I wanted to show a short demo of strongly typing mongoose with unit tests run in nodeunit all using typescript.

## Definitions

First, I have a collection of definition files that represent mongoose types, nodeunit types, and my own document types (in schemaDef.d.ts).

`/def/all.d.ts`

[ts]  
///\<reference path="./mongoose.d.ts"/\>  
///\<reference path="./nodeUnit.d.ts"/\>  
///\<reference path="./schemaDef.d.ts"/\>  
[/ts]

The nodeunit definitions `/def/nodeUnit.d.ts`

[ts]  
interface ITest{  
 done(): void;  
 ok(isGood:Boolean, message?:string):void;  
 equal(expected:any, actual:any, message?:string);  
}  
[/ts]

Here are the basic mongoose definitions in `/def/mongoose.d.ts`. I can't guarantee that these types are right, I'm updating them as I go along. I'm inferring the structure from the documentation and personal experimentation. As I encounter new mongoose definitions, I can just add them to the appropriate scope. You can see that I'm also chaining definitions: one definitions functions might return another interface that has other definitions. This makes it really easy to model the fluent api that mongoose exposes.

[ts]  
interface ICallback{  
 callback(error:string, item:any): void;  
}

interface IEmptyCallback{  
 callback() : void;  
}

interface IErrorCallback{  
 callback(item:string) : void;  
}

interface IWhere{  
 equals(value:String):IChainable;  
 gt(value:String):IChainable;  
 lt(value:String):IChainable;  
 in(value:String[]):IChainable;  
}

interface IChainable{  
 exec(item:ICallback) : IChainable;  
 populate(...args: any[]) : IChainable;  
 select(query:string):IChainable;  
 limit(num:Number):IChainable;  
 sort(field:String):IChainable;  
 where(selector:String):IWhere;  
}

interface IMongooseSearchable{  
 findOne(item:any, callback:ICallback) : void;  
 find(id:string, callback?:ICallback) : IChainable;  
 find(propBag:Object, callback?:ICallback) : IChainable;  
 remove(item:any, callback:IErrorCallback) : void;  
}

interface IMongooseBase {  
 save(item: IEmptyCallback) : void;  
 push(item:IMongooseBase):void;  
}  
[/ts]

Here is my test schema definition `/def/schema.d.ts`. Just for the example I only have a user with a name and an id.

[ts]  
interface IUser extends IMongooseBase{  
 \_id: string;  
 name: string;  
}  
[/ts]

## Modeling the Schema

First, I need to model the schema using mongoose. I have a couple of run once variables that are part of the module export. These are things that are not enclosed with an export tag. They're not global since CommonJS encapsulates everything in a module, but they will run once when the module is created. This makes it easier to reference in the dbclass.

To create data with mongoose you need to first instantiate a schema with a property bag. This gives you an instance that you pass to the mongoose model function. The model function gives you a function reference back that you can use to create new model objects. Since we need to be able to new up objects, we can cast it to an anonymous object that has a `new()` function that returns the correct type we want (IUser). Thanks to Bill Ticehurst for answering [my question](http://stackoverflow.com/a/15399536/310196) regarding how this is done. The target type for the user function reference is of `IMongooseSearchable` because we'll be able to do searching/querying on that model using this.

I've also wrapped the creation of a new user in a helper function that does the appropriate casting that I need. This way, later, I can just call "db.newUser()" to create a new model object without having to worry about casting outside of the db class.

[ts]  
///\<reference path='../def/all.ts'/\>  
var mongoose:any = require("mongoose");

// Need to provide the same structure in 'mongoose' style format to define.  
var userSchema = new mongoose.Schema(  
 {  
 name: String  
 });

export var User:IMongooseSearchable = \<{ new() : IUser; }\>mongoose.model('User', userSchema);

export class db{  
 init(dbName:string, ignoreFailures:bool){  
 if(dbName == null){  
 dbName = "test";  
 }

try{  
 mongoose.connect('localhost', dbName);  
 }  
 catch(e){  
 if(!ignoreFailures){  
 throw e;  
 }  
 }  
 }

disconnect(){  
 mongoose.disconnect();  
 }

newUser():IUser{  
 return \<IUser\>new User();  
 }  
}  
[/ts]

## Testing

And now to test it all with with NodeUnit. Tests are run top to bottom so we can create a test fixture setup as the first function. The downside to this is that it counts as a test itself, but the upside is that we can do a single setup and teardown per group. NodeUnit lets you define setup and teardown for each test as well (but I'm not showing it here) by exposing `setUp` and `tearDown` methods on your exported group object.

As an example, my one query test shows a mongoose save, basic find, and a more complex find with a filter. Notice that the user object has a save method on it (because it inherits from `IMongooseBase`, but to query the users we have to reference the function that was given to us by mongoose that represents the schema. From typescripts perspective though, this is all strongly typed since we've hidden away the casting and weirdness inside of the db class.

I've also used the definitions I made for nodeunit to get a strongly typed nodeunit test object. If I needed more properties for assertions I could add them to the definitions file and they'd be available in typescript for me.

[ts]  
///\<reference path="../def/all.d.ts"/\>  
///\<reference path="../storage/schema.ts"/\>

import schema = module("../storage/schema");

var storage = new schema.db();

export var group = {  
 init: (t:ITest) =\>{  
 storage.init("test");  
 t.done();  
 },

test: (t:ITest) =\>{  
 var u = storage.newUser();

u.name = "test";

u.save(()=\> {  
 schema.User.findOne(u.\_id, (err, user) =\> {  
 console.log(user.name);

schema.User.find(u.\_id)  
 .where("\_id").equals(u.\_id)  
 .exec((err, u1) =\> {  
 console.log(u1);

t.equal(u1[0].name, u.name);  
 t.done();  
 });  
 });  
 });  
 },

end: (t:ITest) =\>{  
 storage.disconnect();  
 t.done();  
 }  
}  
[/ts]

## Conclusion

When working with typescript I found myself frequently checking the compiled javascript to make sure it's what I actually wanted it to be. I found this to workflow to work well when using code that you are typing yourself. This way you can validate that what is compiled makes sense.

## edit:

My schema def's have changed since the posting of this post: for my up to date mongoose schema definitions check my [github](https://github.com/devshorts/trakkit/blob/master/def/mongoose.d.ts)

