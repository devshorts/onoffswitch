---
layout: post
title: Adding static typing and scope validation into the language, part 1
date: 2013-03-04 14:52:47.000000000 -08:00
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
  _jetpack_related_posts_cache: a:1:{s:32:"7535db1f00326ea7d941a323347758fc";a:2:{s:7:"expires";i:1559692912;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3128;}i:1;a:1:{s:2:"id";i:2735;}i:2;a:1:{s:2:"id";i:3565;}}}}

permalink: "/2013/03/04/adding-static-typing-and-scope-validation-into-the-language-part-1/"
---
Continuing on my [series](http://onoffswitch.net/tag/language-implementation/) discussing the language I wrote, this next post is going to talk about the basics of static typing and scope rules. So far my language implementation follows very closely to [Parr's](http://www.cs.usfca.edu/~parrt/) examples in his book [Language Implementation Patterns](http://www.amazon.com/Language-Implementation-Patterns-Domain-Specific-Programming/dp/193435645X), which is what gave me the inspiration to do this project.

However, starting here, I began to get confused with Parr's examples. I didn't really see how to translate his examples into real code. On top of that some of his diagrams weren't matching my mental model of what was going on, so that didn't help either. In general, I understood what he was doing, but I didn't want to follow his examples anymore - Parr switched over to using [ANTLR](http://www.antlr.org/) for everything which didn't help me: I wanted to know how to do this _without_ a generator. So I just jumped in and took a stab at it.

For the scope builder (and later the interpreter) I did what I thought made sense. Like I've mentioned before, I haven't formally studied compiler/language design, so if a solution seems naive or egregiously wrong take it with a grain of salt (but leave a comment so I can learn how to do it better).

Anyways, this is where the project started to take off because I got to make real language decisions. The first two things I decided on was that this language would have [static typing](http://en.wikipedia.org/wiki/Type_system) and [forward reference](http://en.wikipedia.org/wiki/Forward_declaration) support. I, personally, prefer static typed languages to dynamic typed languages because I like compile time checking. I'm not going to deny that dynamic typing isn't helpful in some situations, in fact my interpreter leverages the `dynamic` data type heavily. For fun I did build in some runtime typing into the language (which I'll cover in a later post). The type system I implemented also supports type inference. However, my language is pretty rudimentary and I don't support type promotion, though it wouldn't be hard to add.

For scoping, I wanted to allow forward references for methods and class internals, but not for internal statements. I like forward references because it lets you structure your code by ideas and locality of reference, and not strictly by bottom up execution. I like the freedom that forward references gives you. Also I knew that solving forward references would be more difficult so I decided to tackle the problem.

For the remaining post, I'm going to discuss the groundwork required to dig into more complex parts of the scope builder. As always, if you are curious check out the [github project](https://github.com/devshorts/LanguageCreator) for full source, examples, tests, and comments.

Let's get started.

## Iterating over the syntax tree

In the last post we built a parser that generates an abstract syntax tree representing the execution of our program. Now, we need to iterate over that tree and do meaningful work. However, we're going to be going over this tree a bunch of different times and we want to do different work over it. We could put the tree iteration into the syntax tree, but thats nasty. Instead, I used the [visitor pattern](http://stackoverflow.com/a/255300/310196) to iterate over the tree.

I defined two interfaces. The first, is the definition for the actual visitor.

[csharp]  
public interface IAstVisitor  
{  
 void Visit(Conditional ast);  
 void Visit(Expr ast);  
 void Visit(FuncInvoke ast);  
 void Visit(VarDeclrAst ast);  
 void Visit(MethodDeclr ast);  
 void Visit(WhileLoop ast);  
 void Visit(ScopeDeclr ast);  
 void Visit(ForLoop ast);  
 void Visit(ReturnAst ast);  
 void Visit(PrintAst ast);  
 void Visit(ClassAst ast);  
 void Visit(ClassReference ast);  
 void Visit(NewAst ast);  
 void Visit(TryCatchAst ast);

void Start(Ast ast);  
}  
[/csharp]

This interface will accept different syntax trees and depending on which overloaded `Visit` function is called, the visitor will know how to iterate (and do work) on that particular syntax tree.

The next interface is applied to the abstract base class `Ast`, which means that all syntax tree classes have to implement it. This interface forces the ast classes to "accept" a visitor.

[csharp]  
interface IAcceptVisitor  
{  
 void Visit(IAstVisitor visitor);  
}  
[/csharp]

Below is the class used to represent an expression. Notice its visit method. All the classes have exactly the same visit method. You have to do it this way because if the visit method was in the base class, the visitor wouldn't be able to figure out which class was trying to be visited (since they all inherit from `Ast`).

[csharp highlight="18,19,20,21"]  
public class Expr : Ast  
{  
 public Ast Left { get; private set; }

public Ast Right { get; private set; }

public Expr(Token token) : base(token)  
 {  
 }

public Expr(Ast left, Token token, Ast right)  
 : base(token)  
 {  
 Left = left;  
 Right = right;  
 }

public override void Visit(IAstVisitor visitor)  
 {  
 visitor.Visit(this);  
 }

public override AstTypes AstType  
 {  
 get { return AstTypes.Expression; }  
 }  
}  
[/csharp]

Now all we need to do to iterate over the tree is create an actual visitor. As an example, here is the function invoke overloaded `visit` method on a visitor called `PrintAstVisitor` that iterates the syntax tree and writes the tree representation to the console. To iterate over other parts of the tree we tell the tree to visit itself using this visitor (with `this`).

[csharp]  
public class PrintAstVisitor : IAstVisitor  
{  
 // ... other functions ....

public void Visit(FuncInvoke ast)  
 {  
 Console.WriteLine("Function invoke AST");

ast.FunctionName.Visit(this);

ast.Arguments.ForEach(arg =\> arg.Visit(this));  
 }

// ... other functions ....  
[/csharp]

When we are ready to actually iterate over the full tree, we need to hand off the root of the syntax tree to the visitor and tell it to start.

[csharp]  
var program = @"int x = 1;";

var ast = new LanguageParser(new Lexer(program)).Parse();

new PrintAstVisitor().Start(ast);  
[/csharp]

## Symbols, Types, and Scopes?

The goal of the [scope builder visitor](https://github.com/devshorts/LanguageCreator/blob/master/Lang/Visitors/ScopeBuilderVisitor.cs) is to associate types to symbols and make sure that they are visible in the right scope. We all intuitively know what symbols are, we work with them all the time.

[code]  
int x = 1  
[/code]

`x`, here, is a symbol. It also has a type of `int`. Some types are built into the language, like `int`, `string`, `void`, etc. Some types are user defined (like classes or structs). The scope builder is going to figure out which expressions are which types and tag each syntax tree node with its type.

The builder also starts to do some syntax validation for us. It's responsible for making sure our assignments, declarations, and invocations all make sense. One of the builders job is to make sure we can't reference undefined values. For example, this is invalid:

[code]  
int x = y  
int y = 0;  
[/code]

This is because `x` uses `y` before `y` is defined. The scope builder is also responsible for validating static typing: we want invalid type assignment to be prevented

[code]  
void x = 1;  
[/code]

If we defined x to be a void then we certainly shouldn't be able to assign 1 to a void. The scope builder should throw an exception and prevent us from doing this. By preventing us from doing this we eliminate having to find out at runtime that our code is bogus. Though, as the language "designer" here, if we wanted to let this happen we totally could! It's really up to your language implementation as to how you deal with what this means.

On top of all of this, like mentioned above, the scope builder is responsible for validating that symbols are visible only where they should be. The code below should be invalid because `y` is declared in an inner scope not visible to the print statement. Some languages don't do it this way (javascript/actionscript), but this drives me nuts, so I made sure to do it for my own language.

[code]  
int x = 1;  
{  
 int y = 2;  
}

print y;  
[/code]

The scope builder sounds like it does a lot of work (and it does), but most of these things are intertwined and the underlying code is short and sweet so there isn't too much overlapping of concerns in the same class.

## Scopes

Scopes are easily represented by a stack. Each time you encounter a new scope block (like the inner scoped `y` above) all you need to do is push a new scope onto your scope stack. Symbols below the current scope on the stack are visible if you are in that scope, so `x` would be visible to `y`, but symbols are not allowed to be visible up the stack from the bottom.

To support classes I had one scope stack for the global space. This scoped items in what I considered "main", or really any inline code that is not within a class. I also had a scope stack for each class I encountered. This is because you don't want classes to be able to see symbols in the global scope. Imagine this scenario:

[csharp]  
class foo{  
 int value = x;  
}

int x = 0;  
[/csharp]

This should be invalid because `foo` shouldn't be able to see `x`. _[Note: classes get a little weird, because you need to define the class declaration in the global scope (so you can instantiate the class), but everything inside the class is in its own scope. There are also other issues with classes that I'll talk about in a later post.]_

A scope, then, is nothing more than a bag of symbols and a reference to its parent scope (below it in the scope stack).

For example, a basic `Scope` object in my language looks like this. Notice that a scope contains a reference to its parents scope (line 5). This will get set automatically by the `ScopeStack` I'll show next. It's important to understand that a scope will have access to all elements below it via its parent. You can see that logic in the `Resolve` function which checks if a symbol is in the current scopes dictionary, and if not, tries the parent scope.

[csharp]  
public class Scope : IScopeable\<Scope\>  
{  
 public Dictionary\<string, Symbol\> Symbols { get; set; }

public Scope EnclosingScope { get; private set; }

public List\<IScopeable\<Scope\>\> ChildScopes { get; private set; }

public Scope()  
 {  
 Symbols = new Dictionary\<string, Symbol\>();

ChildScopes = new List\<IScopeable\<Scope\>\>(64);  
 }

public void SetParentScope(Scope scope)  
 {  
 EnclosingScope = scope;  
 }

public void Define(Symbol symbol)  
 {  
 Symbols[symbol.Name] = symbol;  
 }

public Symbol Resolve(String name)  
 {  
 Symbol o;  
 if (Symbols.TryGetValue(name, out o))  
 {  
 return o;  
 }

if (EnclosingScope == null)  
 {  
 return null;  
 }

return EnclosingScope.Resolve(name);  
 }  
}  
[/csharp]

To help with setting the parent scope each time we pushed a new scope onto the stack, I created a generic scope stack that facilitated pushing and popping scopes, as well as auto-linked scope references. This way when I access a scope reference I can go down its stack:

[csharp]  
public class ScopeStack\<T\> where T : class, IScopeable\<T\>, new()  
{  
 private Stack\<T\> Stack { get; set; }

public T Current { get; set; }

public ScopeStack()  
 {  
 Stack = new Stack\<T\>();  
 }

public void CreateScope()  
 {  
 var parentScope = Current;

if (Current != null)  
 {  
 Stack.Push(Current);  
 }

Current = new T();

Current.SetParentScope(parentScope);

if (parentScope != null)  
 {  
 parentScope.ChildScopes.Add(Current);  
 }  
 }

public void PopScope()  
 {  
 if (Stack.Count \> 0)  
 {  
 Current = Stack.Pop();  
 }  
 }  
}  
[/csharp]

And `IScopeable` is

[csharp]  
public interface IScopeable\<T\> where T : class, new()  
{  
 void SetParentScope(T scope);  
 List\<IScopeable\<T\>\> ChildScopes { get; }  
}  
[/csharp]

The concept of a stack based symbol visibility control will get re-used again when we discuss memory spaces (i.e. where a value is in memory). Memory and scope are very similar, but not the same. You'll see why later, but needless to say the scope container class gets re-used a few times in different contexts.

## Types

Every symbol has a type. Let me show you what my type definition looks like:

[csharp]  
public interface IType  
{  
 String TypeName { get; }  
 ExpressionTypes ExpressionType { get; }  
 Ast Src { get; set; }  
}  
[/csharp]

`ExpressionTypes` is an enum of expression types. I use this for type checking later. The `Src` property holds a reference to the syntax tree that generated this type. For example, if I have a class called `foo` I will create a type whose `TypeName` is `foo` and it's `Src` property will point to its `ClassAst` reference. Later I can get info about the source syntax tree just from the type.

I only have two kinds of types in the entire system.

- **Built in types**. These are types like `int`, `string`, etc. They all share the same class, and the only thing that is different about them is the `ExpressionType` enum they hold indicating which built in type they are.
- **User defined types**. These are classes that the user defines.

To make a symbol is easy. Depending on the syntax tree type I can create the proper symbol type based on its token type.

[csharp]  
public static IType CreateSymbolType(Ast astType)  
{  
 if (astType == null)  
 {  
 return null;  
 }

Func\<IType\> op = () =\>  
 {  
 switch (astType.Token.TokenType)  
 {  
 case TokenType.Int:  
 return new BuiltInType(ExpressionTypes.Int);  
 case TokenType.Float:  
 return new BuiltInType(ExpressionTypes.Float);  
 case TokenType.Void:  
 return new BuiltInType(ExpressionTypes.Void);  
 case TokenType.Infer:  
 return new BuiltInType(ExpressionTypes.Inferred);  
 case TokenType.QuotedString:  
 case TokenType.String:  
 return new BuiltInType(ExpressionTypes.String);  
 case TokenType.Word:  
 return new UserDefinedType(astType.Token.TokenValue);  
 case TokenType.True:  
 case TokenType.False:  
 return new BuiltInType(ExpressionTypes.Boolean);  
 case TokenType.Method:  
 return new BuiltInType(ExpressionTypes.Method);  
 }  
 return null;  
 };

var type = op();

if (type != null)  
 {  
 type.Src = astType;  
 }

return type;  
}  
[/csharp]

## Symbols

A symbol is a pairing of a symbol name and a symbol type. This means a variable named `x` declared as type `int` now gets those two values paired together:

[csharp]  
public class Symbol : Scope  
{  
 public String Name { get; private set; }  
 public IType Type { get; private set; }

public Symbol(String name, IType type)  
 {  
 Name = name;  
 Type = type;  
 }

public Symbol(String name)  
 {  
 Name = name;  
 }  
}  
[/csharp]

But wait, symbols are also scopes? They can be. A class symbol is also a scope and when we're within a class we'll want to define it's symbols within it's own personal space (like I mentioned earlier with the separate scope stacks). Having a symbol also be a scope makes life really easy when you try and figure out where things are.

The scope builder is complicated, but it follows the same basic pattern

- If the current symbol is being declared (variable declaration, method declaration argument definitions, etc) then define the symbol in the appropriate scope.
- If we are referencing the symbol, resolve it and make sure it's been declared and is visible in the scope we expect

## Defining a symbol

A simplified version of the variable declration visit function looks like this:

_[Note: it's simplified because we need to handle variables that are declared without values, as well as type inferred values, and method assignments, and partial functions, and validating left and right hand side type assignments, etc. There is a lot to it, to be covered later. If you're curious check the github]_

[csharp]  
public void Visit(VarDeclrAst ast)  
{  
 if (ast.DeclarationType != null)  
 {  
 var symbol = ScopeUtil.DefineUserSymbol(ast.DeclarationType, ast.VariableName);

DefineToScope(symbol);

ast.AstSymbolType = symbol.Type;  
 }  
}  
[/csharp]

The declaration type is a syntax tree representing the declaration of the variable (int, string, user defined, whatever), and the name is an expression representing a word that the variable is called.

Then we create symbol type and the actual symbol

[csharp]  
public static Symbol DefineUserSymbol(Ast ast, Ast name)  
{  
 IType type = CreateSymbolType(ast);

return new Symbol(name.Token.TokenValue, type);  
}  
[/csharp]

Then we define the symbol in the current scope

[csharp]  
private void DefineToScope(Symbol symbol)  
{  
 Current.Define(symbol);  
}  
[/csharp]

And finally, we assign the current ast node it's expression type

[csharp]  
ast.AstSymbolType = symbol.Type;  
[/csharp]

This way each syntax tree can track what type it is. We can use this information to do static type checking later (basically validate that the left hand side and right hand side have the same, or promotable, types).

## Resolving a symbol

Resolving is the other way around.

Here is a simplified version of the visitor for an expression. It first visits the left, then the right. If the left and right are null then we have a leaf in our syntax tree and we need to resolve what it is. If it's a built in type, like an integer, it'll get defined as a symbol, otherwise if it's a word (a user defined variable) it'll get resolved:

[csharp]  
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
}  
[/csharp]

Which calls

[csharp]  
/// \<summary\>  
/// Creates a type for built in types or resolves user defined types  
/// \</summary\>  
/// \<param name="ast"\>\</param\>  
/// \<returns\>\</returns\>  
private IType ResolveOrDefine(Expr ast)  
{  
 if (ast == null)  
 {  
 return null;  
 }

switch (ast.Token.TokenType)  
 {  
 case TokenType.Word: return ResolveType(ast);  
 }

return ScopeUtil.CreateSymbolType(ast);  
}  
[/csharp]

We've already seen `ScopeUtil.CreateSymbolType`. Here is a simplified version of `ResolveType`

[csharp]  
private IType ResolveType(Ast ast)  
{  
 Symbol symbol = Current.Resolve(ast);  
 if(symbol != null){  
 return symbol.Type;  
 }

throw new InvalidSyntax("Cannot resolve {0}", ast.Token.TokenValue);  
}  
[/csharp]

It's a simplified version because it doesn't handle forward references. I'll discuss that in a later post.

Remember the scope rules above. If a variable isn't visible in a scope, either because it's undefined, or because it's not visible, resolving the type will fail.

You'll notice the `SetScope` function. Every syntax tree gets assigned a reference to the scope they came from. This way everyone knows who can see what.

## Conclusion

At this point we have a very basic way of defining symbols, types, and scopes. We've also gone over a way to validate simple scoping rules. There is a lot more to the scope builder. Now that there is some infrastructure in the next post I'll dig deeper into the scope builder and discuss how we can fix the forward reference problem and do type inference. Still to come in the scope builder is determining return types and handling how to build partial functions.

