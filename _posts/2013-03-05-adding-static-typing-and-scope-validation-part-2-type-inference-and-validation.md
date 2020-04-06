---
layout: post
title: 'Adding static typing and scope validation, part 2: type inference and validation'
date: 2013-03-05 15:53:11.000000000 -08:00
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
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561041570;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3016;}i:1;a:1:{s:2:"id";i:2735;}i:2;a:1:{s:2:"id";i:3128;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/03/05/adding-static-typing-and-scope-validation-part-2-type-inference-and-validation/"
---
This post continues my [series](http://onoffswitch.net/tag/language-implementation/) describing how I solved certain problems while creating a [toy programming language](https://github.com/devshorts/LanguageCreator). Today I'll discuss static typing and type inference.

## Tracking types

To have static typing, each syntax tree node needs to track what kind of type it is. Integers are integers, words are resolved user defined types, quoted strings are strings. But terminals are not the only nodes with types. Each syntax trees type is derived from its terminals. For example the expression syntax tree that represents the following:

[csharp]  
(1 + 2) == 5  
[/csharp]

Is actually this:

[code]  
 ==  
 / \  
 + 5  
 / \  
1 2  
[/code]

We have a tree that has an expression on the left, and a literal on the right. We need to know what the expression on the lefts type is before we can do anything. Since the scope builder is depth first and each syntax tree's type is composed of its internal tree types, we'll be guaranteed that by the time we evaluate the type of the `==` tree we'll know that the left hand side is an integer type.

Even though the left hand side expression is an integer, and the right hand side literal is an integer, the expression is held together with an `==` token which means it will be a boolean type. Anything that uses this expression further up the tree can now know that this expression is of type boolean. This is useful info because we probably want to validate that, for example, `if` statements predicates only have boolean types. Same with `while` loops, and parts of a `for` loop, and `else`, etc.

## Type Assignments

For each expression, while resolving types, I set the current expression trees type to be derived from its branches. If we have a leaf, then either resolve the type (if it is a user defined variable) or create a type describing it:

_[Note: when types are resolved, how symbols are created, and solving forward references is coming in [part 3](http://onoffswitch.net/adding-static-typing-and-scope-references-part-3-solving-forward-references/)]_

[csharp highlight="23"]  
public void Visit(Expr ast)  
{  
 if (ast.Left != null)  
 {  
 ast.Left.Visit(this);  
 }

if (ast.Right != null)  
 {  
 ast.Right.Visit(this);  
 }

SetScope(ast);

if (ast.Left == null && ast.Right == null)  
 {  
 ast.AstSymbolType = ResolveOrDefine(ast);  
 }  
 else  
 {  
 if (ResolvingTypes)  
 {  
 ast.AstSymbolType = GetExpressionType(ast.Left, ast.Right, ast.Token);  
 }  
 }  
}  
[/csharp]

The expression visit function calls a helper method that takes the left and right trees as well as the current token

[csharp]  
/// \<summary\>  
/// Determines user type  
/// \</summary\>  
/// \<param name="left"\>\</param\>  
/// \<param name="right"\>\</param\>  
/// \<param name="token"\>\</param\>  
/// \<returns\>\</returns\>  
private IType GetExpressionType(Ast left, Ast right, Token token)  
{  
 switch (token.TokenType)  
 {  
 case TokenType.Ampersand:  
 case TokenType.Or:  
 case TokenType.GreaterThan:  
 case TokenType.Compare:  
 case TokenType.LessThan:  
 case TokenType.NotCompare:  
 return new BuiltInType(ExpressionTypes.Boolean);

case TokenType.Method:  
 case TokenType.Infer:  
 if (right is MethodDeclr)  
 {  
 return new BuiltInType(ExpressionTypes.Method, right);  
 }

return right.AstSymbolType;  
 }

if (!ResolvingTypes && (left.AstSymbolType == null || right.AstSymbolType == null))  
 {  
 return null;  
 }

if (!TokenUtil.EqualOrPromotable(left.AstSymbolType.ExpressionType, right.AstSymbolType.ExpressionType))  
 {  
 throw new Exception("Mismatched types");  
 }

return left.AstSymbolType;  
}  
[/csharp]

Each expression knows what kind of type it should have (based on the infix token) and can validate that its left and right branches match accordingly. Other parts of the scope builder who use these expressions can now determine their types as well (such as function invokes, method declarations, return statements, etc) since the expression is tagged with a type. Anything that can be used as part of another statement needs to have a type associated to it.

## Type inference and validation

Type inference now is super easy. If the left hand side is a type of `var` we don't try to do anything with it, we'll just give it the same type that the right hand side has. For example, in the variable declaration syntax tree visitor I have a block that looks kind of like this:

[csharp]  
// if its type inferred, determine the declaration by the value's type  
if (ast.DeclarationType.Token.TokenType == TokenType.Infer)  
{  
 ast.AstSymbolType = ast.VariableValue.AstSymbolType;

var symbol = ScopeUtil.DefineUserSymbol(ast.AstSymbolType, ast.VariableName);

DefineToScope(ast, symbol);  
}  
[/csharp]

Type validation is also easy, all we have to do is check if the right hand side is assignable to the left hand side. To type check any other (non-expression) syntax trees, like `print`, we can validate that we are expecting the right type (for `print` we don't want to print a void value). Each tree contains its type information as the `AstSymbolType` property that has an enum describing its type that we can use for type checking.

## Inferring method return types

In my language I decided I would allow functions to be declared with a `var` type inferred return type. This means I had to infer its type from its return value. This isn't quite as easy as asking the method declaration tree where its return value is yet, since you have to go find it in the tree. What I did to find it was, while iterating over the source tree, keep track of if I'm inside of a method. The scope builder keeps a single heap allocated property called

[csharp]  
private MethodDeclr CurrentMethod { get; set; }  
[/csharp]

Each time I encounter a method declaration (either by an anonymous lambda or a class method or an inline method), I update this property. I also keep track of what the previous method was on the stack. This way as the scope builder iterates through the tree it always knows what is the current method. When a method is done iterating it'll set `CurrentMethod` to the last method it knew about (or null if there was no method it was inside of)

To help with tracking return statements, I've also added some extra metadata to the `MethodDeclr` AST so every method declaration can now access the syntax tree that represents its return statement directly:

[csharp]  
public class MethodDeclr : Ast  
{  
 /// ...

public ReturnAst ReturnAst { get; private set; }

/// ...  
}  
[/csharp]

During the course of tree iteration, if there is a `return` statement we'll end up hitting the `ReturnAst` visit method and we can tag the current method's return statement with it:

[csharp]  
public void Visit(ReturnAst ast)  
{  
 if (ast.ReturnExpression != null)  
 {  
 ast.ReturnExpression.Visit(this);

ast.AstSymbolType = ast.ReturnExpression.AstSymbolType;

CurrentMethod.ReturnAst = ast;  
 }  
}  
[/csharp]

Here is my `MethodDeclr` visit method

[csharp]  
public void Visit(MethodDeclr ast)  
{  
 var previousMethod = CurrentMethod;

CurrentMethod = ast;

var symbol = ScopeUtil.DefineMethod(ast);

Current.Define(symbol);

ScopeTree.CreateScope();

ast.Arguments.ForEach(arg =\> arg.Visit(this));

ast.Body.Visit(this);

SetScope(ast);

if (symbol.Type.ExpressionType == ExpressionTypes.Inferred)  
 {  
 if (ast.ReturnAst == null)  
 {  
 ast.AstSymbolType = new BuiltInType(ExpressionTypes.Void);  
 }  
 else  
 {  
 ast.AstSymbolType = ast.ReturnAst.AstSymbolType;  
 }  
 }  
 else  
 {  
 ast.AstSymbolType = symbol.Type;  
 }

ValidateReturnStatementType(ast, symbol);

ScopeTree.PopScope();

CurrentMethod = previousMethod;  
}  
[/csharp]

Let's trace through it:

- Lines 3 and 5. Keep track of the previous method we came from (if any), and set the current method to point to the syntax tree we're on
- Lines 7 and 9. Create a method symbol and define it in the current scope. This makes the method visible to anything in the same scope
- Line 11. Create a new scope. All internal method arguments and statements are inside of their own scope
- Lines 13 and 15. Visit the arguments and body
- Line 17. Set the method syntax tree's scope to the current (so now it points to the current scope and can access this later).
- Lines 19 through 29. If the method is a type inferred return type, set the methods type to be the same as the return statement types. If there isn't a return statement, make it a void.
- Line 32. If its not a type inferred return type, set the type of the method syntax tree to be the same as the symbol we created for it. 
- Line 35. `ValidateReturnStatement` checks to make sure that the declared type matches the return statement type. If we declared a function to return string but we are returning a user object, then that's going to generate a compiler error. \>/li\>
- Line 37. We're done with this method scope so pop the scope off the stack
- Line 39. Reset the previous current method to the tracking property

Now we've properly validated the method declaration type with its return value, and if we needed to type inferred the method from its return statement. So, as an example, we can support something like this:

[csharp]  
[Test]  
public void TestTypeInferFunctionReturn()  
{  
 var test = @"

var func(){  
 return 'test';  
 }  
 ";

var ast = (new LanguageParser(new Lexers.Lexer(test)).Parse() as ScopeDeclr);

var function = ast.ScopedStatements[0] as MethodDeclr;

new InterpretorVisitor().Start(ast);

Console.WriteLine("Inferred return type: " + function.AstSymbolType.ExpressionType);

Console.WriteLine("Original declared return expression type: " + function.MethodReturnType);  
}  
[/csharp]

Which prints out

[code]  
Inferred return type: String  
Original declared return expression type: Infer: var  
[/code]

## Conclusion

Now the language has static typing and type inference. It doesn't have type promotion, but now that all the types are defined and propagated through the tree its easy to add. Next I'll talk about forward references and how I solved that problem. As always, check the [github](https://github.com/devshorts/LanguageCreator) for full source, examples, and tests.

