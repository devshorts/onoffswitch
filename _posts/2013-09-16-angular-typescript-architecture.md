---
layout: post
title: Angular with typescript architecture
date: 2013-09-16 08:00:32.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- angularjs
- architecture
- typescript
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561704481;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3710;}i:1;a:1:{s:2:"id";i:3295;}i:2;a:1:{s:2:"id";i:3452;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/09/16/angular-typescript-architecture/"
---
Bear with me, this is going to be a long post.

For the past several months I've been working on a production single page application written with AngularJS and TypeScript, and I wanted to share how myself and my team have architected the application. In case you were wondering, the app was written using typescript 0.8.3 and not 0.9.1 (which is out now with generics).

In general, I was really unhappy with a lot of the AngularJS examples I had found on the internet when researching how to structure the application, since they all looked flimsy and poorly constructed. While the examples found online were easy to read and follow, they clearly wouldn't work with an actual large application.

I had several goals:

- I didn't want to have to constantly register directives/filters/controllers/etc with Angular everytime I added something
- I didn't want to have to update my main index.html page with any references to new files as I worked on them
- I wanted to avoid string typing as much as possible by centralizing all string references to angular components
- I wanted everything testable
- I didn't want to inline directive html templates, I wanted them in in separate html components
- I wanted one file for each class
- I wanted everything strongly typed

Anything less than these requirements, I felt, would compromise maintainability and extensibility for the future, especially if the app had more than one developer working on it.

## Folder structure

Starting off, my folder structure looks like this:

[code]  
app  
├──components  
├──css  
├──img  
├──js  
 ├──common  
 ├──controllers  
 ├──data  
 ├──locale  
 ├──def  
 ├──local  
 ├──vendor  
 ├──directives  
 ├──filters  
 ├──models  
 ├──services  
 ├──app.ts  
 ├──\_all.d.ts  
├──locales  
├──partials  
├──tests  
 ├──js  
 ├──unitTests  
 ├──controllres  
 ├──directives  
 ├──filters  
 ├──models  
 ├──services  
 ├──e2e tests  
├──index.html  
[/code]

- components - This is where all the html templates for directives go
- css - All scss files and the final compiled app.css which is what is linked to from index.html
- img - All statically served image files
- js - All typescript files broken down into individually subsections. Also the definitions folder `def` will be seperated out from any local custom definitions if necessary, and all vendor definitions (such as angular, rxjs, jquery, etc). Also, `app.ts` (which will be discussed later) is the main app bootloader. `_all.d.ts` is an aggregate typescript def file that has all the references to the .ts files in the application
- locales - This is where source .properties files go that can be used to [auto generate typescript locale data](https://github.com/devshorts/TypescriptLocaleGenerator)
- partials - This is where main angular page entrypoints go. These are the initial partials that are loaded as part of an `ng-view` change
- tests - all unit tests, and e2e tests (using angular e2e scenario testing)

## Wrapping Angular in OO

There's no reason you can't wrap the angularJS object initializers that are used to define directives, filters, controllers, etc, into strongly typed local classes. My team and I built a bunch of templates to use with IntelliJ Idea which made creating new directives/controllers/filters/services/etc extremely easy.

## Controllers

Let me demonstrate an example controller:

[ts]  
/// \<reference path="../\_all.d.ts" /\>

module devshorts.app.controllers {

import sharedModel = common.model;

export interface ITest extends ng.IScope{

}

export class Test extends ControllerBase {

public static $inject:string[] = [  
 sharedModel.AngularGlobal.$SCOPE  
 ];

constructor(private $scope:ITest) {  
 super(arguments, Test.$inject);  
 }  
 }  
}  
[/ts]

There are a few things going on here. First, there is a single aggregate definitions file that is always included. This means we don't need to pollute each file with definitions. The `_all.d.ts` is also auto generated using a dependency builder we wrote (posted on my [github](https://github.com/devshorts/TypeScript-Dependency-Builder)). It finds all `.d.ts` files based on a dependency configuration and auto populates the all.d.ts for you.

Second, notice that the controller scope is typed with an interface. By making sure we use an interface we can guarantee we won't try to access a field in the UI that we didn't explicity expose.

Third, since for controllers I'm using the `$inject` annotation, we're making sure to not hardcode any angular strings. Everything is centralized in an `AngularGlobal` class:

[ts]  
export class AngularGlobal {  
 public static $SCOPE = "$scope";  
 public static $COOKIE\_STORE = "$cookieStore";  
 public static NG\_COOKIES = "ngCookies";  
 ... etc ...  
 }  
[/ts]

Also, you'll notice that in the constructor of the controller there is a call to the base class with the inject arguments. The base class is going to validate that the arguments passed to the controller match the items in the $inject array. This way we can get fail fast runtime errors if we ask to inject something, but didn't wire up the constructor right:

[ts]  
export class ControllerBase {

definedArguments(args:any):string[] {  
 var functionText = args.callee.toString();  
 var foundArgs = /\(([^)]+)/.exec(functionText);  
 if (foundArgs[1]) {  
 return foundArgs[1].split(/\s\*,\s\*/);  
 }

return [];  
 };

constructor(args:any, injection:string[]){

var expectedInjections = \_.zip(this.definedArguments(args), injection);

\_.each(expectedInjections, val =\> {  
 var injectionId = val[0];  
 var argument:string = val[1];  
 if(argument == null){  
 throw "missing injection id. Argument for " + injectionId + " is undefined. Make sure to add the ID as part of the $inject function";  
 }  
 })  
 }

[/ts]

So, why does moving the constructor to its own class work? Well, look at what angular wants for a controller:

[javascript]  
myApp.controller('GreetingCtrl', ['$scope', function($scope) {  
 $scope.greeting = 'Hola!';  
}]);  
[/javascript]

And look at what typescript will generate for this class:

[javascript highlight="8,9,10,11,15"]  
var devshorts;  
(function (devshorts) {  
 (function (app) {  
 (function (controllers) {  
 var sharedModel = common.model;  
 var Test = (function (\_super) {  
 \_\_extends(Test, \_super);  
 function Test($scope) {  
 \_super.call(this, arguments, Test.$inject);  
 this.$scope = $scope;  
 }  
 Test.$inject = [  
 sharedModel.AngularGlobal.$SCOPE  
 ];  
 return Test;  
 })(ControllerBase);  
 controllers.Test = Test;  
 })(app.controllers || (app.controllers = {}));  
 var controllers = app.controllers;  
 })(devshorts.app || (devshorts.app = {}));  
 var app = devshorts.app;  
})(devshorts || (devshorts = {}))  
[/javascript]

Disregarding all the wrapping for namespaces, you can see that the constructor for Test is just a function that takes a $scope. So, it's the exact same thing. Now you can wire up your routes in such a way to pair partials with controllers like this:

[ts]  
$routeProvider.when('/test',  
 this.getRoute(relativePath("partials/test.html"),  
 devshorts.app.controllers.Test));  
[/ts]

## Auto registration of services, directives, filters, and models

For our purposes, we did something slightly different with services, models, directives and filters. The reason being that controllers are always "registered" with angular via the routing. However, services, models, directives, and filters are usually registered with separate angular modules that the main app depends on. In this scenario, since it's common to create lots of directives, etc, we didn't want to have to manually register anything.

Let me show the model template we have:

[ts]  
/// \<reference path="../\_all.d.ts" /\>

module devshorts.app.models {  
 'use strict';

export interface ITestModel {

}

export class TestModel implements ITestModel {

public static ID:string = "TestModel";

public static injection():any[] {  
 return [ AngularGlobal.HTTP,  
 httpService =\> new TestModel(httpService) ];  
 }  
 }  
}  
[/ts]

Here there is a static `ID` which is the name of the model, and a static `injection` function that returns the array notation for injection. In the array notation the last line is the function that gets called by angular with the relevant injectable types. We preferred array notation vs $inject notation since array notation is safe to minimize. The name `httpService` at this point doesn't matter, since we asked angular for it using the static `AngularGlobal` class discussed above.

The `ID` and `injection` functions are important, and I'll show why in a second.

Below is how we've bootstrapped angular in the index.html page

[javascript]  
$(document).ready(function(){  
 var main = new devshorts.app.Main(ipadGlobals.available);

// when all is done, execute bootstrap angular application  
 angular.bootstrap(document, [NG\_GLOBAL.APP\_NAME]);  
 });  
[/javascript]

But what is this main class? This is a class that handles all the registration, routing, and other AngularJS setup we need.

[ts]  
export class Main implements IMain {

public app:ng.IModule = angular.module(NG\_GLOBAL.APP\_NAME,  
 [  
 NG\_GLOBAL.APP\_DIRECTIVES,  
 NG\_GLOBAL.APP\_SERVICES,  
 NG\_GLOBAL.APP\_MODELS,  
 NG\_GLOBAL.APP\_PROVIDERS,  
 NG\_GLOBAL.APP\_FILTERS,  
 'ui.directives',  
 'ngMobile'  
 ]);

public directives:ng.IModule = angular.module(NG\_GLOBAL.APP\_DIRECTIVES, []);

public services:ng.IModule = angular.module(NG\_GLOBAL.APP\_SERVICES, []);

public models:ng.IModule = angular.module(NG\_GLOBAL.APP\_MODELS, [sharedModel.AngularGlobal.NG\_COOKIES]);

public providers:ng.IModule = angular.module(NG\_GLOBAL.APP\_PROVIDERS,[]);

public filters:ng.IModule = angular.module(NG\_GLOBAL.APP\_FILTERS, []);

constructor(private isAvailable:bool) {  
 this.route(this.getApp());

this.wireFactories();

this.configureProviders();

this.configureHttpInterceptors(this.getApp());  
 }

... other methods ...  
[/ts]

Again, the app name and other dependency names are all centralized in a static class. The interesting part here is the `wireFactories` method which looks like this:

[ts]  
wireFactories(){  
 this.wireServices();  
 this.wireDirectives();  
 this.wireModels();  
 this.wireFilters();  
}  
[/ts]

Lets look at `wireModels`

[ts]  
wireModels(){  
 this.wire(devshorts.app.models, this.models.factory);  
}  
[/ts]

And finally looking at `wire`

[ts]  
/\*\*\*  
 \* We are simulating doing something like this:  
 \*  
 \* this.directives.directive(directives.DynamicView.ID, directives.DynamicView.injection());  
 \*  
 \* The "this.directives.directive" is the function we want to call on which is the registration function  
 \*  
 \* Since the ID and injection function are statically defined in our classes and MUST be defined for  
 \* our angular injection to work, we can type them temporarily here in this function  
 \*  
 \* This way if we add new items to any namespace that have an injection function and an ID we will  
 \* automatically register them to the right angular module \*  
 \*  
 \* @param namespace  
 \* @param registrator  
 \* @param byPass  
 \*/  
 wire(namespace:any, registrator:(string, Function) =\> ng.IModule , byPass?:(s) =\> bool){  
 for(var key in namespace){  
 try{  
 if(byPass != null && byPass(key)) {  
 continue;  
 }

var injector = \<IInjectable\>(namespace[key]);

if(injector.ID && injector.injection){  
 registrator(injector.ID, injector.injection());  
 }  
 }  
 catch(ex){  
 console.log(ex);  
 }  
 }  
}  
[/ts]

Since namespaces in javascript are just objects with properties, we can make an assumption that anything under the `devshorts.app.models` namespace is a model if it has a static `ID` and a static `injection` function. If it does, then we can register that class with the correct angular module.

Now we never have to worry about wiring up directives, filters, models, or services since at runtime they are auto wired for us. This gives application development a more native feel, just by adding the file means it exists and is available for injection.

## Merging the files

I've read about people promoting writing everything in javascript into one file, since they don't want to have the overhead of loading hundreds of .js files on load. I vehemently disagree here. From a development standpoint you should have one class per file. It makes it easier to split up your code, move things around, and to properly organize your application. However, in a production environment you aboslutely should distribute a single merged file. But that's trivial. I linked to it earlier on, but the dependency tool I had also populates index.html with the appropriate script references for all .js files it finds (that are configured for it to find). If you are interested, go check out the [typescript dependency builder](https://github.com/devshorts/TypeScript-Dependency-Builder) (which I will probably blog about again later).

Assuming that your index.html page has all the relevant js files in the head, then it's easy to write a simple script to merge all the files into one and serve that up when the application is loaded in production. In debug mode, you can serve up all the independent files, it's just a matter of toggling your node config or your web.config (in an asp.net application).

As an example, here is an F#/C# MSBuild task class to do that for you:

[csharp collapse="true"]  
public class JsMerger : Task  
{  
 public string IndexPage { get; set; }  
 public List\<string\> JsFiles { get; set; }  
 public String OutputFile { get; set; }

public override bool Execute()  
 {  
 try  
 {  
 if (File.Exists(OutputFile))  
 {  
 File.Delete(OutputFile);  
 }

var filesToProcess = GetFilesToProcess()  
 .Where(NotOutputOrMinFile)  
 .Select(f =\> new JsData  
 {  
 FileName = f,  
 FileContents = File.ReadAllText(f)  
 });

using (var output = new StreamWriter(OutputFile))  
 {  
 foreach (var file in filesToProcess)  
 {  
 output.WriteLine("/\*!" + Path.GetFileName(file.FileName) + "\*/");  
 output.WriteLine(file.FileContents);  
 output.WriteLine(Environment.NewLine);  
 }  
 }  
 return true;  
 }  
 catch (Exception ex)  
 {  
 Console.WriteLine(ex);  
 return false;  
 }  
 }

private bool NotOutputOrMinFile(string f)  
 {  
 var path = f;

var check = OutputFile.Replace(".js", "");

return (path != check + ".js") && (path != check + ".min.js");  
 }

private IEnumerable\<string\> GetFilesToProcess()  
 {  
 if (!String.IsNullOrEmpty(IndexPage))  
 {  
 foreach (var jsFile in JsRetriever.getJsFiles(IndexPage))  
 {  
 yield return jsFile;  
 }  
 }

if (JsFiles != null && JsFiles.Count \> 0)  
 {  
 foreach (var f in JsFiles)  
 {  
 yield return f;  
 }  
 }  
 }  
}  
[/csharp]

Which calls into the following F# script extractor

[fsharp collapse="true"]  
namespace MergeJsFiles

open HtmlAgilityPack  
open System.Text.RegularExpressions  
open System.IO  
open System

module JsRetriever =

let stripHtml (text:string) =  
 try  
 let mutable target = text

let regex = [  
 "\<script\s\*", "";  
 "\"?\s\*type\s\*=\s\*\"\s\*text/javascript\s\*\"\s\*", "";  
 "\</script\>", "";  
 "src\s\*=\s\*", ""  
 "\"", "";  
 "\>", "";  
 "\</",""  
 "\<",""

]

for (pattern, replacement) in regex do  
 target \<- Regex.Replace(target,pattern,replacement).Trim()

target  
 with  
 | ex -\>  
 Console.WriteLine ("Error handling " + text + ", " + ex.ToString())  
 ""

let convertToAbsolute parent path =  
 try  
 Path.Combine(Path.GetDirectoryName(parent), path) |\> Path.GetFullPath  
 with  
 | ex -\>  
 Console.WriteLine ("Error handling " + path)  
 ""

let endsOn ext file =  
 Path.GetExtension(file) = ext

let getJsFiles (defaultAspxPath:string) =  
 let doc = new HtmlDocument()

doc.Load defaultAspxPath

doc.DocumentNode.SelectNodes "/html/head/script/@src"  
 |\> Seq.map (fun i -\> i.OuterHtml)  
 |\> Seq.map stripHtml  
 |\> Seq.map (convertToAbsolute defaultAspxPath)  
 |\> Seq.filter (endsOn ".js")

[/fsharp]

## Conclusion

Organizing an application is hard, and everyone has their own style, but proper application organization and architecture is critical to being able to scale your codebase. Also, sometimes to make the development experience pleasant, you need to invest the time to build tools to automate boring tasks for you. Until we spent the time to create the dependency tools, working with angular and typescript was a real headache, but now it's an absolute joy.

There are more things I haven't covered, such as how we dealt with filters, localization, model and service aggregation (for easy injection), and http interceptors but I'll save those for another post.

