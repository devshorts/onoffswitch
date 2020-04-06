---
layout: post
title: A handrolled language parser
date: 2013-03-01 10:40:44.000000000 -08:00
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
- Lexer
- monads
- parser
- projects
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561472777;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4131;}i:1;a:1:{s:2:"id";i:4068;}i:2;a:1:{s:2:"id";i:4365;}}}}

permalink: "/2013/03/01/a-handrolled-language-parser/"
---
In my [previous post](http://onoffswitch.net/building-a-custom-lexer/) about building a custom lexer I mentioned that, for educational purposes, I created a simple [toy programming language](https://github.com/devshorts/LanguageCreator) (still unnamed). There, I talked about building a tokenizer and lexer from scratch. In this post I'll discuss building a parser that is responsible for generating an [abstract syntax tree](http://stackoverflow.com/questions/1721553/how-to-construct-an-abstract-syntax-tree) (AST) for my language. This syntax tree can then be passed to other language components such as a scope and type resolver, and finally an interpreter.

The parser I made is a [recursive descent](http://en.wikipedia.org/wiki/Recursive_descent_parser) [packrat parser](http://stackoverflow.com/a/7141394/310196) that uses backtracking. Short of memoizing found AST, there aren't any other real optimizations. The goal was to create a working parser, not a production parser to distribute or use (or reuse) in any professional sense. Like the lexer, this is an academic exercise to try and hit on some of the points covered by [Terence Parr's](http://www.cs.usfca.edu/~parrt/) [Language Implementation Patterns](http://www.amazon.com/Language-Implementation-Patterns-Domain-Specific-Programming/dp/193435645X) book that I recently finished reading.

I'm not going to cover much language theory because I want to jump into what I did and how it's implemented. There are a lot of resources on context free grammars, LL parsers, left-recursion, ambiguous vs unambiguous grammars, PEG (parsing expression grammars), shift reduce parsing, parse tables, and other subjects related to parsing and language implementation on the internet. I'll leave explanations of those topics for someone else who is more qualified than me. But, if you are interested and are new to it (like myself), starting with Parr's book is a great first step and helps clear up some of the theoretical haze that surrounds a lot of programming language theory.

## Syntax definitions

Every language is defined by a grammar, where the syntax represents the rules of the grammar. The grammar of my language was grown organically, I didn't really go into it with any specific syntax. I knew I wanted to have lambdas, the `var` keyword, and simple stuff like if, else, while, for, function declarations and class declarations. When I wrote the parser I just started with some basic syntax and added to it as I wanted more functionality. I think most people give their languages a bit more thought than I did, but all's well that ends well.

In general, you can represent your grammar using [BNF](http://en.wikipedia.org/wiki/Backus%E2%80%93Naur_Form). Let's define a really simple language:

[code]  
word := [A-z]+  
number := \d+  
operator := + | -  
token := word | number  
expression := token | token operator expression  
variableDeclration = var word = expression  
statement : = variableDeclaration | expression  
ifStatement := if (expression) { statement\* }  
[/code]

What this translates to is:

- **word**. This is any group of characters at least once with no spaces. `foo` for example would be a word
- **number**. This is any number of digits (no decimals). Something like `1234`
- **operator**. This is a plus sign or a minus sign
- **token**. Either a word or a number
- **expression**. This is either a token or a token with an operator followed by an expression. This example here is what is called right recursive, since the expression references itself on the right hand side of the operation. Recursive descent parsers [can't handle left recursion](http://stackoverflow.com/questions/847439/why-cant-a-recursive-descent-parser-handle-left-recursion) since it leads to infinite loops. 
- **variable declaration**. This is the keyword `var` followed by a word followed by an `=` followed by an expression
- **statement**. Either a variable declaration or some expression
- **if statement**. The keyword `if` followed by a `(` followed by an expression, followed by a `)` followed by a `{` followed by one or more statements, followed by a `}`

If you wanted to you, you could feed this general grammar (modified syntactically) to libraries that can auto generate parsers for you, but that's no fun and feels like cheating when you are learning.

## The Parsers Job

The goal of the parser is to take a strongly typed token stream from the lexer, and to create a syntax tree that we can use. A simple way to think about it is that each bullet point in our grammar can be a class. Imagine we have a class called `IfStatement`. It might look something like this:

[csharp]  
public class IfStatement{  
 public Expression Predicate { get; set; }  
 public List\<Statement\> Statements { get; set; }  
}  
[/csharp]

We don't really care about the keyword `if` or any other of the special characters like (, ), {, and }, since the important part is that we now have a class that describes what the if statement meant: a [predicate](http://en.wikipedia.org/wiki/Predicate_(mathematical_logic)), and a list of statements to execute if the predicate is true. This type of class is what is known as a syntax tree. It's a tree because it references other syntax nodes. Here is an image showing what a syntax tree could look like (image taken from [wikipedia](http://en.wikipedia.org/wiki/Abstract_syntax_tree))

![531px-Abstract_syntax_tree_for_Euclidean_algorithm.svg](http://onoffswitch.net/wp-content/uploads/2013/02/531px-Abstract_syntax_tree_for_Euclidean_algorithm.svg_.png)

When you're done parsing, you will have a root node that references the entire structure of your program. Later parts of the language chain (building out scope, types, and interpreting the code) will go through these definitions and actually evaluate what they mean. Those later passes might also add extra metadata to each node, so the AST is like our master repo of program related metadata. For now the goal is to create these classes.

## Capturing Information

Since the parsers goal is to create these classes, it needs to be able to work on an underlying token stream and create meaningful representations from that stream. The parser knows what kinds of syntactical patterns it expects. For example, if we have the following variable declaration and assignment

[code]  
int x = 5;  
[/code]

We can tell it's a variable declaration and assignment because it matches the pattern of

[code]  
valid type  
word  
equals  
valid assignment (an expression maybe or a single token?)  
semicolon  
[/code]

The parsers job is to take those tokens in meaningful orders and create an AST from it. When I say "take", I mean you remove the current token from the head of the stream (or advance the token streams index pointer). Lets say we are parsing that variable declaration above, it might have a token stream that looks like this

[code]  
int (keyword)  
word (x)  
equals (keyword)  
number (5)  
semicolon (keyword)  
[/code]

We see that the head of the stream is a keyword that can be a valid variable type (int), so we can take it off the list and store it. Then we expect the pattern "word", "equals", "expression", "semicolon". We can take them one at a time and while it matches keep on going. Certain items like the semicolon you can trash. It's there to tell the parser when to stop.

## Alternatives

[Sometimes](http://en.wikipedia.org/wiki/LL_parser#LL.281.29_Conflicts), however, you can't determine what an expression will be just by looking at the current token. For example, what does this mean if you only look at the first element?

[code]  
1 + 1  
[/code]

Is it a token of value 1? Or is it an expression of 1 + 1? Obviously it's 1 + 1, but the parser can't always tell. Careful ordering of your parser can avoid most of these ambiguities, but when it can't, you can either peek into the stream (so you see that the next token is a + so that means expression), or simply try alternatives. The first alternative to match wins!

For alternatives, you try first an expression. If that fails, then you try a token. If that fails, then invalid syntax. Remember that I mentioned that my parser is a packrat parser? All this means is that while it's trying alternatives it will cache if it found them. Later, when I actually go and try to take a certain branch I can retrieve the already parsed AST from the cache. This cuts down on a lot of extra work.

## The Token Stream

In the last [post](http://onoffswitch.net/building-a-custom-lexer/) about the lexer, I created a `TokenizableStreamBase` base class that handles basic snapshot/commit/rollback/consume functionality on an input stream. Here I'm going to re-use it and pass it a stream of tokens, instead of a stream of characters. The parser will instantiate this `ParseableTokenStream` class (which subclasses the tokenizable stream base) and use it as it's token stream.

The most basic form of the class is this:

[csharp]  
public class ParseableTokenStream : TokenizableStreamBase\<Token\>  
{  
 public ParseableTokenStream(Lexer lexer) : base (() =\> lexer.Lex().ToList())  
 {  
 }

... implementation ...  
}  
[/csharp]

It takes the lexer, lexes the tokens, and creates an underlying token stream that we can do snapshots on. We also have methods to test if the current item on the stream is a specific token type (defined by a known enum):

[csharp]  
public Boolean IsMatch(TokenType type)  
{  
 return Current.TokenType == type;  
}  
[/csharp]

The parsing stream base also lets us "take" a specific token. If you remember from the last post, all consuming of a lexable item does is advance the internal array index. The important part is that after we `Consume`, we've advanced to the next token in the token stream.

You'll see in my parser that sometimes I use the `Take` return value, and sometimes it's discarded. This is intentional. Even if you don't intend to use a token in the syntax tree (like a semicolon) you still have to acknowledge that it was part of the expected pattern and advance the token stream.

[csharp]  
public Token Take(TokenType type)  
{  
 if (IsMatch(type))  
 {  
 var current = Current;

Consume();

return current;  
 }

throw new InvalidSyntax(  
 String.Format("Invalid Syntax. Expecting {0} but got {1}",  
 type,  
 Current.TokenType));  
}  
[/csharp]

We can also try an alternate route. If the route function returns a non-null syntax tree we'll assume the route succeeded and cache it. Later requests for getting syntax trees at that current index will first check the cache before trying to re-build the tree (if it needs to):

[csharp]  
public Boolean Alt(Func\<Ast\> action)  
{  
 TakeSnapshot();

Boolean found = false;

try  
 {  
 var currentIndex = Index;

var ast = action();

if (ast != null)  
 {  
 found = true;

CachedAst[currentIndex] = new Memo  
 {  
 Ast = ast,  
 NextIndex = Index  
 };  
 }  
 }  
 catch  
 {

}

RollbackSnapshot();

return found;  
}  
[/csharp]

The `CachedAst` field is defined as

[csharp]  
private Dictionary\<int, Memo\> CachedAst = new Dictionary\<int, Memo\>();  
[/csharp]

Where `Memo` is

[csharp]  
internal class Memo  
{  
 public Ast Ast { get; set; }  
 public int NextIndex { get; set; }  
}  
[/csharp]

There are also couple of extra methods that let me try a route, and if it succeeds return it's cached results

[csharp]  
public Ast Capture(Func\<Ast\> ast)  
{  
 if (Alt(ast))  
 {  
 return Get(ast);  
 }

return null;  
}

/// \<summary\>  
/// Retrieves a cached version if it was found during any alternate route  
/// otherwise executes it  
/// \</summary\>  
/// \<param name="getter"\>\</param\>  
/// \<returns\>\</returns\>  
public Ast Get(Func\<Ast\> getter)  
{  
 Memo memo;  
 if (!CachedAst.TryGetValue(Index, out memo))  
 {  
 return getter();  
 }

Index = memo.NextIndex;

return memo.Ast;  
}  
[/csharp]

The underlying stream in my parser is an array, so the inherited `Index` property keeps track of where we are in the stream. When we return a memoized syntax tree, we can seek the stream to the index directly after the last memoized token. This means we can easily jump around in our parser stream. Hopefully this makes sense, because if we returned a cached syntax tree that spanned token items 1 through 15, we should jump immediately to token index 16 and continue parsing from there.

For small parsing this works well, but obviously wouldn't scale with large programs. Still, to change it so that we work on a buffered section of an infinite stream wouldn't be that much work, and in the end wouldn't modify how the actual parser behaves. This is all hidden in the shared base class (so the lexer would also improve).

## Finally, Parsing

First, to tie in the section above, here is the constructor of the [parser](https://github.com/devshorts/LanguageCreator/tree/master/Lang/Parser):

[csharp]  
private ParseableTokenStream TokenStream { get; set; }

public LanguageParser(Lexer lexer)  
{  
 TokenStream = new ParseableTokenStream(lexer);  
}  
[/csharp]

Next, I've defined a few syntax tree classes that the parser will use:

![ast.](http://onoffswitch.net/wp-content/uploads/2013/02/ast..png)

All of the syntax tree containers inherit from the base class `Ast`. This makes working with syntax trees in the parser easy because everything can be passed around as the base, and it means we can extend the metadata that syntax trees have just by adding to the base class. Hopefully most of the class names are self explanatory (if statement, while loop, method declaration, class dereference) just by class name. If you're interested in class details you can go to the [github](https://github.com/devshorts/LanguageCreator/tree/master/Lang/AST) and check them out. Suffice to say that they look kind of like the if statement class I pseudocoded earlier.

As an example, let me show one that I reused a lot. The `ScopeDeclr` AST gets created anytime the parser encounters a `{` followed by some statements, terminated by `}`.

[csharp]  
public class ScopeDeclr : Ast  
{  
 public List\<Ast\> ScopedStatements { get; private set; }

public ScopeDeclr(List\<Ast\> statements) : base(new Token(TokenType.ScopeStart))  
 {  
 ScopedStatements = statements;  
 }

public override void Visit(IAstVisitor visitor)  
 {  
 visitor.Visit(this);  
 }

public override AstTypes AstType  
 {  
 get { return AstTypes.ScopeDeclr; }  
 }  
}  
[/csharp]

`ScopedStatements` is a list of statements that are found in the scoped block.

I used the `ScopeDeclr` syntax tree to hold the root node of the entire application. This is because I considered the global scope (starting at the root) to be, well, a scope. The `ScopeDeclr` also turned out to be extremely useful when building out partial curried functions, a subject I'll cover in the next post about scope and type definitions.

Here is the entrypoint to the parser:

[csharp]  
public Ast Parse()  
{  
 var statements = new List\<Ast\>(1024);

while (TokenStream.Current.TokenType != TokenType.EOF)  
 {  
 statements.Add(ScopeStart().Or(Statement));  
 }

return new ScopeDeclr(statements);  
}  
[/csharp]

The `.Or()` method is an extension method I added inspired by the [maybe monad](http://en.wikibooks.org/wiki/F_Sharp_Programming/Computation_Expressions#Monad_Primer). It returns the first non-null result in a chain of functions.

[csharp]  
public static class Maybe  
{  
 public static TInput Or\<TInput\>(this TInput input, Func\<TInput\> evaluator)  
 where TInput : class  
 {  
 if (input != null)  
 {  
 return input;  
 }

return evaluator();  
 }  
}  
[/csharp]

Lets take a look at what is a `Statement`

[csharp]  
/// \<summary\>  
/// Class, method declaration or inner statements  
/// \</summary\>  
/// \<returns\>\</returns\>  
private Ast Statement()  
{  
 var ast = TokenStream.Capture(Class)  
 .Or(() =\> TokenStream.Capture(MethodDeclaration));

if (ast != null)  
 {  
 return ast;  
 }

// must be an inner statement if the other two didn't pass  
 // these are statements that can be inside of scopes such as classes  
 // methods, or just global scope  
 var statement = InnerStatement();

if (TokenStream.Current.TokenType == TokenType.SemiColon)  
 {  
 TokenStream.Take(TokenType.SemiColon);  
 }

return statement;  
}  
[/csharp]

A statement can either be a class, a method declaration, or an inner statement. I didn't want to need to put semicolons after class and method definitions, so I don't test for a semicolon there. I also made semicolons optional, if we can unambiguously determine the grammar without needing semicolons then great, otherwise we'll use it to terminate a statement if it's there. Though in reality you need to put in semicolons or the parser will barf. Call it a [language quirk](https://www.google.com/search?q=programming+language+quirks).

Here is an inner statement. These are statements I considered valid within scopes such as method declarations, global scope, or inside of classes.

[csharp]  
/// \<summary\>  
/// A statement inside of a valid scope  
/// \</summary\>  
/// \<returns\>\</returns\>  
private Ast InnerStatement()  
{  
 // ordering here matters since it resolves to precedence  
 var ast = TryCatch().Or(ScopeStart)  
 .Or(LambdaStatement)  
 .Or(VariableDeclWithAssignStatement)  
 .Or(VariableDeclrStatement)  
 .Or(GetIf)  
 .Or(GetWhile)  
 .Or(GetFor)  
 .Or(GetReturn)  
 .Or(PrintStatement)  
 .Or(Expression)  
 .Or(New);

if (ast != null)  
 {  
 return ast;  
 }

throw new InvalidSyntax(String.Format("Unknown expression type {0} - {1}", TokenStream.Current.TokenType, TokenStream.Current.TokenValue));  
}  
[/csharp]

Let's check out a few other parsers. Here is how to parse a `new` of the form

[csharp]  
new thing(a, b, c)  
[/csharp]

I explicity didn't put in a semicolon, since semicolons delimit statements, not just expressions.

This gives me a class `NewAst` that has the class name (`thing`) and a list of the arguments (`a`, `b`, and `c`).

[csharp]  
private Ast New()  
{  
 Func\<Ast\> op = () =\>  
 {  
 if (TokenStream.Current.TokenType == TokenType.New)  
 {  
 TokenStream.Take(TokenType.New);

var name = new Expr(TokenStream.Take(TokenType.Word));

List\<Ast\> args = GetArgumentList();

return new NewAst(name, args);  
 }

return null;  
 };

return TokenStream.Capture(op);  
}  
[/csharp]

We test if the current token is of type `TokenType.New` and if so consumes it. Then it expects an expression (the word `thing`), and then gets a comma delimited list of arguments. There's no semicolon because this `new` statement is part of a larger sequence of statements which will contain a reference to this `new` on the tree. We don't really know, or care, if the statement is part of a variable declaration, or a print statement, or a function call, or whatever, as long as its valid in the grammar.

Here is a `while`

[csharp]  
private Ast GetWhile()  
{  
 if (TokenStream.Current.TokenType == TokenType.While)  
 {  
 Func\<WhileLoop\> op = () =\>  
 {  
 var predicateAndStatements = GetPredicateAndStatements(TokenType.While);

var predicate = predicateAndStatements.Item1;

var statements = predicateAndStatements.Item2;

return new WhileLoop(predicate, statements);  
 };

return TokenStream.Capture(op);  
 }

return null;  
}  
[/csharp]

Which leverages the following helper function

[csharp]  
private Tuple\<Ast, ScopeDeclr\> GetPredicateAndStatements(TokenType type)  
{  
 TokenStream.Take(type);

TokenStream.Take(TokenType.OpenParenth);

var predicate = InnerStatement();

TokenStream.Take(TokenType.CloseParenth);

var statements = GetStatementsInScope(TokenType.LBracket, TokenType.RBracket);

return new Tuple\<Ast, ScopeDeclr\>(predicate, statements);  
}  
[/csharp]

Hopefully you can see now how this all continues on. `GetStatementsInScope` pulls all semicolon delimited statements between a left bracket and a right bracket and returns a scope declaration block with them inside.

## Expressions

I wanted to dedicate a specific section on parsing expressions because I struggled with this. These are ones like

[code]  
1 + 1  
(b.x.z \* 2.0)  
(new class()).x == true  
(f + 2) + foo() + 3 + (a - 2 - z)  
[/code]

I'll be truthful here, I didn't think expressions through thoroughly before I started. For every pattern I was able to match I exposed one that I couldn't. At one point I ran into a bunch of left recursion issues. In the end, expressions, as I've "_defined_" them look like this

[code]  
operator = + | - | / | ^ | = | | | == | !=  
terminal = new statement | function call | class dereference | single token  
expression' = terminal operator expression | terminal  
expression = ( expression ) | ( expression ) operator expression | expression'  
[/code]

This was mostly figured out through trial and error, some pen and paper diagrams, extensive unit tests, and a lot of head scratching. Honestly, out of the whole parser this is what took the longest to get right (at least right enough).

What I did to avoid left recursion, I later realized, looks similar to what [wikipedia](http://en.wikipedia.org/wiki/Left_recursion#Removing_left_recursion) suggests, which is to create a new intermediate nonterminal. This is the subset of specific terminals I called `terminal` in the BNF above. So I'm not just matching on _expression operator expression_ since that would recurse endlessly (assuming tail call recursion) or, more likely, just blow up my stack and crash.

The expression parsing code, in the end, matches expressions of the following formats (for example). I made all the examples use a plus sign because I was lazy - any available operator works in any ordering (these cases are from my expression testing unit test)

[csharp]  
1 + 2;  
1 + 2 + 3;  
(1 + 2) + 3;  
(1 + 2 ) + (3 + 4);  
1 + (2 + 3);  
1 + (2 + 3) + 4;  
(1 + 2 + 3 + 4);  
new foo().z + 1;  
a.f().z \* 2.0 + (new foo().x + 2);  
(new foo().z) + 1;  
(f + 2) + foo() + 3 + (a + 2 + z)  
[/csharp]

Which when tested, looks something like this

[csharp]  
SCOPE:  
(Int: 1 Plus: + Int: 2)  
(Int: 1 Plus: + (Int: 2 Plus: + Int: 3))  
((Int: 1 Plus: + Int: 2) Plus: + Int: 3)  
((Int: 1 Plus: + Int: 2) Plus: + (Int: 3 Plus: + Int: 4))  
(Int: 1 Plus: + (Int: 2 Plus: + Int: 3))  
(Int: 1 Plus: + ((Int: 2 Plus: + Int: 3) Plus: + Int: 4))  
(Int: 1 Plus: + (Int: 2 Plus: + (Int: 3 Plus: + Int: 4)))  
([( new Word: foo with args n/a). (Word: z)] Plus: + Int: 1)  
([( Word: a). (call Word: f with args ). (Word: z)] Asterix: \* (Float: 2.0 Plus: + ([( new Word: foo with args n/a). (Word: x)] Plus: + Int: 2)))  
([( new Word: foo with args n/a). (Word: z)] Plus: + Int: 1)  
((Word: f Plus: + Int: 2) Plus: + (call Word: foo with args Plus: + (Int: 3 Plus: + (Word: a Plus: + (Int: 2 Plus: + Word: z)))))  
[/csharp]

Like the other parse functions, this one returns an `Ast` and does some basic alternative checking. The `new` test, on line 3, isn't part of `IsValidOperand` because I re-use `IsValidOperand` elsewhere.

[csharp]  
private Ast Expression()  
{  
 if (IsValidOperand() || TokenStream.Current.TokenType == TokenType.New)  
 {  
 return ParseExpression();  
 }

switch (TokenStream.Current.TokenType)  
 {  
 case TokenType.OpenParenth:

Func\<Ast\> basicOp = () =\>  
 {  
 TokenStream.Take(TokenType.OpenParenth);

var expr = Expression();

TokenStream.Take(TokenType.CloseParenth);

return expr;  
 };

Func\<Ast\> doubleOp = () =\>  
 {  
 var op1 = basicOp();

var op = Operator();

var expr = Expression();

return new Expr(op1, op, expr);  
 };

return TokenStream.Capture(doubleOp)  
 .Or(() =\> TokenStream.Capture(basicOp));

default:  
 return null;  
 }  
}  
[/csharp]

What we're doing here is splitting up the operation into 3 different sections

- Terminal. This is the first statement. If it's a terminal parse and return. Terminals aren't just single tokens, they are terminal expression types (like `new`, function calls, single operands, simple operations like _operand operator operand_, etc)
- Expressions inside of parenthesis. If we have something like `(1 + 1)`, take the parenthesis out and parse the expression.
- Expressions inside of parenthesis, followed by an operator, followed by an expression. If we have `(1 + 1) + (1 - a)`, or `(1 + 1) + 2 + 3`, take the first section, then the operator, then try the next section

If we have a valid left operand we can parse a basic expression that is of the form

[code]  
terminal | terminal operator expression  
[/code]

This is right recursive! Sweet, no recursion issues. If you didn't catch why earlier, check out [this](http://stackoverflow.com/questions/847439/why-cant-a-recursive-descent-parser-handle-left-recursion) stack overflow question. The ordering of parsing here matters, I am parsing from most terms to least terms. If I switched the order (terminal first, then expression), the parser would break since we'd run into the alternative issue I mentioned in a section above.

Here is how I parsed the basic expression defined in the above BNF

[csharp]  
private Ast ParseExpression()  
{  
 Func\<Func\<Ast\>, Func\<Ast\>, Ast\> op = (leftFunc, rightFunc) =\>  
 {  
 var left = leftFunc();

if (left == null)  
 {  
 return null;  
 }

var opType = Operator();

var right = rightFunc();

if (right == null)  
 {  
 return null;  
 }

return new Expr(left, opType, right);  
 };

Func\<Ast\> leftOp = () =\> op(ExpressionTerminal, Expression);

return TokenStream.Capture(leftOp)  
 .Or(() =\> TokenStream.Capture(ExpressionTerminal));  
}  
[/csharp]

Where `IsValidOperand` is

[csharp]  
private bool IsValidOperand()  
{  
 switch (TokenStream.Current.TokenType)  
 {  
 case TokenType.Int:  
 case TokenType.QuotedString:  
 case TokenType.Word:  
 case TokenType.True:  
 case TokenType.Float:  
 case TokenType.Nil:  
 case TokenType.False:  
 return true;  
 }  
 return false;  
}  
[/csharp]

And `ExpressionTerminal` is

[csharp]  
private Ast ExpressionTerminal()  
{  
 return ClassReferenceStatement().Or(FunctionCallStatement)  
 .Or(New)  
 .Or(SingleToken);  
}  
[/csharp]

## Testing the parser

At this point everything is set up! I didn't cover all of the parser functions but they are pretty similar in nature. Anyways, lets do a few tests

[csharp]  
[Test]  
public void TestSimpleAst()  
{  
 var test = @"x = 1;";

var ast = new LanguageParser(new Lexers.Lexer(test)).Parse() as ScopeDeclr;

var expr = (ast.ScopedStatements[0] as Expr);

Assert.IsTrue(expr.Left.Token.TokenType == TokenType.Word);  
 Assert.IsTrue(expr.Right.Token.TokenType == TokenType.Int);  
 Assert.IsTrue(ast.Token.TokenType == TokenType.ScopeStart);  
}  
[/csharp]

And something more complicated

[csharp]  
[Test]  
public void AstWithExpression2()  
{  
 var test = @"int z = 1;  
 {  
 int y = 5 + 4;  
 }  
 x = 1 + 2 ^ (5-7);";

var ast = new LanguageParser(new Lexers.Lexer(test)).Parse() as ScopeDeclr;

Assert.IsTrue(ast.ScopedStatements.Count == 3);  
 Assert.IsTrue(ast.ScopedStatements[0] is VarDeclrAst);  
 Assert.IsTrue(ast.ScopedStatements[1].Token.TokenType == TokenType.ScopeStart);  
 Assert.IsTrue(ast.ScopedStatements[2] is Expr);

Console.WriteLine(ast);  
}  
[/csharp]

Let me print out the above test string representation:

[code]  
SCOPE:  
 Declare Word: z as Int: int with value Int: 1  
 SCOPE:  
 Declare Word: y as Int: int with value (Int: 5 Plus: + Int: 4)  
 (Word: x Equals: = (Int: 1 Plus: + (Int: 2 Carat: ^ (Int: 5 Minus: - Int: 7))))  
[/code]

And this insane block of gibberish.

[csharp]  
[Test]  
public void FunctionTest()  
{  
 var test = @"void foo(int x, int y){  
 int x = 1;  
 var z = fun() -\> {  
 zinger = ""your mom!"";  
 someThing(a + b) + 25 - (""test"" + 5);  
 };  
 }

z = 3;

int testFunction(){  
 var p = 23;

if(foo){  
 var x = 1;  
 }  
 else if(faa){  
 var y = 2;  
 var z = 3;  
 }  
 else{  
 while(1 + 1){  
 var x = fun () -\>{  
 test = 0;  
 };  
 }

if(foo){  
 var x = 1;  
 }  
 else if(faa){  
 var y = 2;  
 var z = 3;  
 }  
 else{  
 for(int i = 0; i \< 10; i = i + 1){  
 var x = z;  
 }  
 }  
 }  
 }";

var ast = new LanguageParser(new Lexers.Lexer(test)).Parse() as ScopeDeclr;

Assert.IsTrue(ast.ScopedStatements.Count == 3);  
 Assert.IsTrue(ast.ScopedStatements[0] is MethodDeclr);  
 Assert.IsTrue(ast.ScopedStatements[1] is Expr);  
 Assert.IsTrue(ast.ScopedStatements[2] is MethodDeclr);  
}  
[/csharp]

Well, you get the idea.

## Conclusion

One thing I didn't like about how my parser turned is that the entire parsing code ended up in one class. I couldn't think of a clean way to separate all this out, since we have lots of mutual recursion going on. Each function needed to know about the other one. I thought about passing around the lexer as a state (functional style) but when I tried that it ended up messier than I had hoped.

Anyways, it's a lot to take in, and I showed a bunch of code, some of which is out of context. Make sure to go to the [github](https://github.com/devshorts/LanguageCreator) if you want to poke around some more. After doing the parser, I wouldn't do one again by hand. This is pretty tedious and it took me two frustrating weeks to get it all right (thanks for bearing with me [Tracy](https://twitter.com/nightCheese2)!). This is why you would absolutely use a pre-rolled lexing/parsing solution to get you to your abstract syntax trees. [Here](http://en.wikipedia.org/wiki/Comparison_of_parser_generators) is a list of parser generators, and I'm sure there are tons not listed. Lots of smart people have built these and will save you plenty of time in the long run.

Like the exercise with the lexer, I'm glad I did it and I have a much better appreciation for libraries that do this all for you.

The next step (and post) will be to determine proper scoping rules, static typing (and type validation), and add some extra neat features like partial functions to our language. This is going to be done using a scope builder that will use the visitor pattern to iterate over our syntax tree. Having the syntax tree means we can finally start doing interesting stuff and making language decisions.

## Disclaimer

If I got something wrong in the post please let me know! Like I've mentioned before I'm new at this and just sharing my findings.

