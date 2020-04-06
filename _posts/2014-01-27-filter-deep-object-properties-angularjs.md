---
layout: post
title: Filter on deep object properties in angularjs
date: 2014-01-27 08:00:25.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- angularjs
- JavaScript
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561519402;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4028;}i:1;a:1:{s:2:"id";i:3710;}i:2;a:1:{s:2:"id";i:265;}}}}

permalink: "/2014/01/27/filter-deep-object-properties-angularjs/"
---
AngularJS provides a neat way of filtering arrays in an `ng-repeat` tag by piping elements to a built in `filter` filter which will filter based on a predicate. Doing it this way you can filter items based on a function, or an expression (evaluated to a literal), or by an object.

When filtering by an object you can pass in a javascript object that represents key value items that should match whats in the backing data element. For example:

[js]  
$scope.elements = [{ foo: "bar" }, { foo: "biz" }]

--

\<div ng-repeat="foos in elements | filter: { foo: "bar" }"\>  
 {{ foos.foo }} matches "bar"  
\</div\>  
[/js]

Here I filtered all the objects whose foo property matches the value "bar". But what if I have a non-trivial object? Something with lots of nested objects? I found that passing in the object to the filter was both unweidly, and error prone. I wanted something simple where I could write out how to dereference it, like `obj.property.otherItem.target`.

Thankfully this is pretty easy to write:

[js]  
function initFilters(app){  
 app.filter('property', property);  
}

function property(){  
 function parseString(input){  
 return input.split(".");  
 }

function getValue(element, propertyArray){  
 var value = element;

\_.forEach(propertyArray, function(property){  
 value = value[property];  
 });

return value;  
 }

return function (array, propertyString, target){  
 var properties = parseString(propertyString);

return \_.filter(array, function(item){  
 return getValue(item, properties) == target;  
 });  
 }  
}  
[/js]

And can be used in your html like this:

[js]  
\<ul\>  
 only failed: \<input type="checkbox"  
 ng-model="onlyFailed"  
 ng-init="onlyFailed=false"/\>

\<li ng-repeat="entry in data.entries | property:'test.status.pass':!onlyFailed"\>  
 \<test-entry test="entry.test"\>\</test-entry\>  
 \</li\>  
\</ul\>  
[/js]

