---
layout: post
title: 'Adding static typing and scope references, part 3: solving forward references'
date: 2013-03-07 08:00:45.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- language implementation
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561171017;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3016;}i:1;a:1:{s:2:"id";i:3164;}i:2;a:1:{s:2:"id";i:3161;}}}}

permalink: "/2013/03/07/adding-static-typing-and-scope-references-part-3-solving-forward-references/"
---
In an [earlier post](http://onoffswitch.net/adding-static-typing-and-scope-validation-into-the-language-part-1/) I gave a brief overview of the scope builder and its jobs. There I mentioned that supporting forward references required some extra work. In this post I'll talk more about how I solved forward references.

Here is what I mean by forward references. `func` is declared after it's being referenced

```csharp
  
string item = func();

string func(){  
 return "yes";  
}

print item;  

```

If we iterate over the program only once from the top down using our visitor pattern based scope builder, when we try and resolve the `func` method invocation symbol we'll get an error (it hasn't been defined yet).

Remember that when things are declared (such as methods, classes, or variables) we create a symbol (with a type) in the current scope tree. Later, when we are referencing them, we need to resolve that symbol. Resolution both validates that we can properly see the symbol and gives us information about that symbol (such as its type, which we can use for static typing). This is important because without forward reference support, we can't tell what the return type of the `func()` call is, so we can't validate that that it needs to be a string (or something that can be promoted to a string).

## Maintaining scope references

But what if instead of iterating over the tree once, we iterate over it twice? The idea is that the first pass defines all your types, and the second pass can now resolve all the types.

Since I'm going to iterate over the tree twice, I need a way to persist scopes across each iteration regardless of the context of the scope builder. For that, I added a property on the syntax tree base class to track what its current scope is. Every syntax tree will contain a reference to the scope it came from.

```csharp
  
public abstract class Ast : IAcceptVisitor  
{  
 // ...

public Scope CurrentScope { get; set; }

// ...  
}  

```

The trick here is that I'm leveraging the fact that, in C#, classes are reference types. For those more familiar with C++ this would be like sharing a pointer across multiple syntax trees all pointing to the same scope object. This means that when one syntax tree defines symbols to the current scope, any other syntax elements that have already been processed, and share the same pointer, will also see the newly added elements. Because of this, when the scope builder is all done, each node in the tree knows what else it can see.

## The first pass 

Going with the example above

```csharp
  
string item = func();  

```

In the first pass we'll do something like this

- Create user defined symbol `item` with type `string` (a built in type)
- Visit the variable declaration value (a function invoke)
- In the function invoke visit method, resolve the symbol `func`. But, this resolving fails and returns null. We can't resolve `func` yet because we don't know who or what it is. We can't really do any type validation here yet because the symbol is null
- Define `func` as a method symbol in the same scope (global) that has a return type of type `string`. Creating symbols was covered in the previous post
- Set the variable declaration asts scope to the current so it can see other symbols in its scope

Now the scope definitions table looks like this, with all fields defined

```
  
item - string  
func - method (returns string)  

```

## Persist the scope on the syntax tree

Let's look at how to set the scope on an ast

```csharp
  
private void SetScope(Ast ast)  
{  
 if (ast.CurrentScope == null)  
 {  
 ast.CurrentScope = Current;

ast.Global = Global;  
 }

if (ast.CurrentScope != null && ast.CurrentScope.Symbols.Count \< Current.Symbols.Count)  
 {  
 ast.CurrentScope = Current;  
 }

if (ast.Global != null && ast.Global.Symbols.Count \< Global.Symbols.Count)  
 {  
 ast.Global = Global;  
 }  
}  

```

There are a couple conditions here. First, if the current scope object is null, set it. This should happen on the first pass. Also attach the global scope so that each syntax tree (even if it comes from a class) can look into the global scope. I mentioned in an earlier post that the syntax tree is the master repo of program related metadata and we're starting to see that here. The two other if statements are safety checks to make sure that if the current scope has more symbols defined than what the syntax tree sees, make sure to update the syntax trees internal scope blocks. This way we always make sure to have the most up to date symbol defintions.

Once we have the scope set, we can do the second pass on the syntax tree. This will create a new scope builder and we'll run the tree back through it. It'll do basically the same thing as the first pass, except this time, it's going to look in the previously persisted scope for the variable declaration tree.

## The second pass and resolving symbols

First let's look at how the first and second passes are invoked. I invoke them as part of the `Start` method of the interpreter, so this happens automatically anytime I interpret a program:

```csharp
  
var scopeBuilder = new ScopeBuilderVisitor();

var resolver = new ScopeBuilderVisitor(true);

scopeBuilder.Start(ast);

resolver.Start(ast);  

```

The `true` passed to the visitor's constructor tells the second pass resolver to resolve types. With regards to our example that I showed above

```csharp
  
string item = func();

string func(){  
 return "yes";  
}

print item;  

```

The first time when we hit the variable declaration we'll go through the variable declaration part of the visitor:

```csharp
  
public void Visit(VarDeclrAst ast)  
{  
 // ...

if (ast.VariableValue != null)  
 {  
 ast.VariableValue.Visit(this);  
 }

SetScope(ast);

// ...  
}  

```

Where the variable value is a syntax tree representing a method invocation (`func`). When the variable value is visited, the function invoke visitor method will do this:

```csharp
  
public void Visit(FuncInvoke ast)  
{  
 // ...

if(ResolvingTypes)  
 {  
 ast.AstSymbolType = ResolveType(ast.FunctionName, ast.CurrentScope);  
 }

SetScope(ast);

// ...  
}  

```

Notice that second parameter, the `ast.CurrentScope`, which is the trees persisted scope. This is the scope reference from the FIRST pass, and we only ever set it on the first pass, meaning that at this point the variables are all declared.

`ResolveType` then looks like this.

```csharp
  
/// \<summary\>  
/// Resolve the target ast type from the current scope, OR give it a scope to use.  
/// Since things can be resolved in two passes (initial scope and forward reference scope)  
/// we want to be able to pass in a scope override. The second value is usually only ever used  
/// on the second pass when determining forward references  
/// \</summary\>  
/// \<param name="ast"\>\</param\>  
/// \<param name="currentScope"\>\</param\>  
/// \<returns\>\</returns\>  
private IType ResolveType(Ast ast, Scope currentScope = null)  
{  
 var scopeTrys = new List\<Scope\> { currentScope, ast.CurrentScope };

try  
 {  
 return Current.Resolve(ast).Type;  
 }  
 catch (Exception ex)  
 {  
 try  
 {  
 return ast.CallingScope.Resolve(ast).Type;  
 }  
 catch  
 {  
 foreach (var scopeTry in scopeTrys)  
 {  
 try  
 {  
 if (scopeTry == null)  
 {  
 continue;  
 }

var resolvedType = scopeTry.Resolve(ast);

var allowedFwdReferences = scopeTry.AllowedForwardReferences(ast);

if (allowedFwdReferences ||  
 scopeTry.AllowAllForwardReferences ||  
 resolvedType is ClassSymbol ||  
 resolvedType is MethodSymbol)  
 {  
 return resolvedType.Type;  
 }  
 }  
 catch  
 {

}  
 }  
 }  
 }

if (ResolvingTypes)  
 {  
 if (ast.IsPureDynamic)  
 {  
 return new BuiltInType(ExpressionTypes.Inferred);  
 }

throw new UndefinedElementException(String.Format("Undefined element {0}",  
 ast.Token.TokenValue));  
 }

return null;  
}  

```

The basic gist is first try to resolve the type from the current scope (which gets reset every time the scope builder runs over the tree. So even on the second pass, the first line will fail, and thats what we want).

`Current` contains the following:

```
  
item - string  

```

`func` isn't defined yet. `Current` gets reset each time the scope builder is instantiated, so even though this is the second pass this scope reference still fails. That's OK. If we are resolving types (i.e. the second pass), we can try an alternate scope. We can either pass in an alternate scope, or also use the syntax trees previous scope. For each alternate scope we try and find the symbol we want. If we use the previously constructed ast scope, which contains

```
  
item - string  
func - method - string  

```

it would have the `func` symbol defined as a method with return string. Other alternate scopes can be like the global scope, which we would try if we are within the context of a class. This makes sense because if you are doing this:

```csharp
  
class first{  
 var secondInstance = new second();  
}

class second{  
 int x = 0;  
}  

```

When you are inside of the `first` class and vising the variable declaration, the `second` class is defined in the global scope, NOT the current class scope. This means resolving the class name from the classes scope will fail, but if we give it an alternate (the global scope) then it will succeed.

Anyways, if we resolve as a forward reference, then we need to determine if we're _allowed_ to see it as a forward reference. Everything at this point is resolvable (since we've populated the persisted scope reference from the first pass). Method symbols and class symbols are always allowed, and scopes can optionally define forward reference allowance if they need to. But regular statements aren't allowed to be forward references. We still want this to fail:

```csharp
  
int x = y;  
int y = 0;  

```

After all of the resolving, if we still don't have the type, or we weren't allowed to see it as a forward reference, then we've encountered an error and we have to bail.

_[Note: I mentioned that I implemented runtime dynamic typing and you can see part of that at the end of the resolve types function. If the syntax tree is deemed to be "purely dynamic" then it will never fail when resolving a type. The type will then be figured out at runtime, which means you get no static analysis on the type ahead of time]_

## Updating the AST scope reference

If we did manage to resolve the type, the next step is to go back and update the type reference in the asts scope declaration:

```csharp
  
private void DefineToScope(Ast ast, Symbol symbol)  
{  
 if (ast.CurrentScope != null && ast.CurrentScope.Symbols.ContainsKey(symbol.Name))  
 {  
 Symbol old = ast.CurrentScope.Resolve(symbol.Name);  
 if (old.Type == null)  
 {  
 ast.CurrentScope.Define(symbol);  
 }  
 }

Current.Define(symbol);  
}  

```

Notice that first if statement. What this is saying is that if we have some defined symbol in the scope, but that defined symbol does NOT have a type, then redefine the symbol. All this is doing is filling in the gaps in the `ast.CurrentScope` so we can use that as the master scope repository later in the interpreter.

## Up next

Coming up next I'll discuss how I added in partial functions into the scope builder. After that it'll be time to jump into the interpreter.

## Disclaimer

As with the other posts, I haven't formally studied language implementation or compiler design. This series is just documenting how I solved problems I encountered while trying to make a working [toy language](https://github.com/devshorts/LanguageCreator) using [Terence Parr](http://www.cs.usfca.edu/~parrt/)'s [Language Implementation Patterns](http://www.amazon.com/Language-Implementation-Patterns-Domain-Specific-Programming/dp/193435645X) book. If there are things you'd like to share (or correct) leave a comment and let me know!

