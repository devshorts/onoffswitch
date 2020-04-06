---
layout: post
title: Implementing partial functions
date: 2013-03-11 08:00:18.000000000 -07:00
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
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559795293;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3565;}i:1;a:1:{s:2:"id";i:4028;}i:2;a:1:{s:2:"id";i:2020;}}}}

permalink: "/2013/03/11/implementing-partial-functions/"
---
This next section I had a lot of fun with, and originally I didn't plan on implementing it at all. The only reason I did it is because I had a stroke of genius while in the shower one morning. Today, I'm going to talk about how I supported partial functions in my toy programming language.

First let's look at what a partial function looks like in my language. I took an F# approach where any function whose argument count is less than the declared count becomes a new function (even though F# functions are curried by default but mine are not). For example, in [ML type notation](http://en.wikibooks.org/wiki/Standard_ML_Programming/Types) you could have a function of type

[fsharp]  
'a -\> 'b -\> 'c  
[/fsharp]

Which means that it takes something of type a and type b as arguments, and returns a type c. If we pass this function only a `'a` then it'll return to us a new function of type

[fsharp]  
'b -\> 'c  
[/fsharp]

Since the first argument has been captured.

Here is an example from one of my unit tests to drive the point home

[csharp]  
void func(string printer, int y){  
 print printer;  
 print y;  
}

var partial = func('guy');

int x = 1;

partial(x);

partial(2);

var otherPartial = func('girl');

otherPartial(3);  
[/csharp]

The `partial` variable is a function who now expects only 1 argument (since the first string argument to `func` was already applied). This outputs

[code]  
guy  
1  
guy  
2  
girl  
3  
[/code]

## Manipulating the syntax tree

If a function invoke AST has fewer arguments than the corresponding method symbol it's paired with then it needs to be partially applied. _[Note: I should mention that partial functions and currying are[not technically](http://stackoverflow.com/a/10443057/310196) the same, even though in practice the terminology is interchangeable. I only realized this after I named all the functions, so even if things are called "curry" this and "curry" that, it really means "partially apply"]_

[csharp]  
public void Visit(FuncInvoke ast)  
{  
 if (ast.CallingScope != null)  
 {  
 ast.Arguments.ForEach(arg =\> arg.CallingScope = ast.CallingScope);  
 }

ast.Arguments.ForEach(arg =\> arg.Visit(this));

SetScope(ast);

var functionType = Resolve(ast.FunctionName) as MethodSymbol;

if (functionType != null && ast.Arguments.Count \< functionType.MethodDeclr.Arguments.Count)  
 {  
 var curriedMethod = CreateCurriedMethod(ast, functionType);

curriedMethod.Visit(this);

var methodSymbol = ScopeUtil.DefineMethod(curriedMethod);

Current.Define(methodSymbol);

ast.ConvertedExpression = curriedMethod;  
 }  
 else if(ResolvingTypes)  
 {  
 ast.AstSymbolType = ResolveType(ast.FunctionName, ast.CurrentScope);  
 }  
}  
[/csharp]

What I end up doing is creating new hidden syntax tree that represents a closed on syntax tree. In the example above we'll end up with an anonymous function that looks like this:

[csharp]  
void anonymous1(int y){  
 string printer = 'guy';  
 print printer;  
 print y;  
}  
[/csharp]

The captured argument becomes a variable declaration of the same name as the argument name with the value that was captured! At this point the rest of the code works as is, because what does it care if it was defined this way by the user or by tree manipulation? It doesn't. There is one caveat. I created the concept of "converted expressions" for syntax trees. This means that what used to be a type inferred variable declaration, should now be treated as a method declaration. This is important. `partial` (in our original example) is no longer a variable, it is now a method. When I go to interpret this syntax tree, and use type definitions, I first need to check if the tree has been converted to something else (via the converted expression property).

## Building out the partial function

Building the actual partial function isn't that hard.

1. For each captured argument create a variable declaration syntax tree representing the argument name and the argument value
2. Resolve the argument symbol. If the passed in argument was a user defined value, we need to validate that the type of that user defined value matches the expected type of the argument we are capturing. 
3. Add the new variable declaration to a list
4. Create a new method where the variable declarations we just made are prepended to the original methods body

From here on out we can treat this method just like any other regularly declared method. The `MethodSymbol` type of the original method has a reference to its source syntax tree. This way we can get access back to the data that this method symbol contains

[csharp]  
private LambdaDeclr CreateCurriedMethod(FuncInvoke ast, MethodSymbol functionType)  
{  
 var srcMethod = functionType.MethodDeclr;

var fixedAssignments = new List\<VarDeclrAst\>();

var count = 0;  
 foreach (var argValue in ast.Arguments)  
 {  
 var srcArg = srcMethod.Arguments[count] as VarDeclrAst;

var token = new Token(srcArg.DeclarationType.Token.TokenType, argValue.Token.TokenValue);

var declr = new VarDeclrAst(token, srcArg.Token, new Expr(argValue.Token));

// if we're creating a partial using a variable then we need to resolve the variable type  
 // otherwise we can make a symbol for the literal  
 var newArgType = argValue.Token.TokenType == TokenType.Word ?  
 ast.CurrentScope.Resolve(argValue).Type  
 : ScopeUtil.CreateSymbolType(argValue);

// create a symbol type for the target we're invoking on so we can do type checking  
 var targetArgType = ScopeUtil.CreateSymbolType(srcArg.DeclarationType);

if (!TokenUtil.EqualOrPromotable(newArgType, targetArgType))  
 {  
 throw new InvalidSyntax(String.Format("Cannot pass argument {0} of type {1} to partial function {2} as argument {3} of type {4}",  
 argValue.Token.TokenValue,  
 newArgType.TypeName,  
 srcMethod.MethodName.Token.TokenValue,  
 srcArg.VariableName.Token.TokenValue,  
 targetArgType.TypeName));  
 }

fixedAssignments.Add(declr);

count++;  
 }

var newBody = fixedAssignments.Concat(srcMethod.Body.ScopedStatements).ToList();

var curriedMethod = new LambdaDeclr(srcMethod.Arguments.Skip(ast.Arguments.Count).ToList(), new ScopeDeclr(newBody));

SetScope(curriedMethod);

return curriedMethod;  
}  
[/csharp]

