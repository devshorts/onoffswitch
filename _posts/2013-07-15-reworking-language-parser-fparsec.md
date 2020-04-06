---
layout: post
title: Reworking my language parser with fparsec
date: 2013-07-15 08:00:21.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- abstract syntax trees
- F#
- fparsec
- fsharp
- parsing
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560221353;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:3016;}i:2;a:1:{s:2:"id";i:4365;}}}}

permalink: "/2013/07/15/reworking-language-parser-fparsec/"
---
Since I was playing with fparsec last week, I decided to redo (or mostly) the parser for my homebrew language that I've previously posted about. Using fparsec made the parser surprisingly succinct and expressive. In fact I was able to do most of this in an afternoon, which is impressive considering[my last C# attempt](http://onoffswitch.net/a-handrolled-language-parser/) took 2 weeks to hammer out.

As always, it starts with the data

[fsharp]  
type Op =  
 | Plus  
 | Minus  
 | GreaterThan  
 | LessThan  
 | Mult  
 | Divide  
 | Carrot

type Ast =  
 | Statement of Ast  
 | Expression of Ex  
 | Function of string option \* Argument list option \* Ast  
 | Scope of Ast list option  
 | Class of Ex \* Ast  
 | Conditional of Ex \* Ast \* Ast option  
 | WhileLoop of Ex \* Ast  
 | ForLoop of Ast \* Ex \* Ex \* Ast  
 | Call of string \* Argument list option  
 | Assign of Ex \* Ex  
and Ex =  
 | Single of Ast  
 | Full of Ex \* Op \* Ex  
 | Float of float  
 | Int of int  
 | Literal of string  
 | Variable of string  
and Argument =  
 | Element of Ex  
[/fsharp]

## Operators

Parsing operators is trivial

[fsharp]  
let plus = pstring "+" \>\>% Plus

let minus = pstring "-" \>\>% Minus

let divide = pstring "/" \>\>% Divide

let mult = pstring "\*" \>\>% Mult

let carrot = pstring "^" \>\>% Carrot

let gt = pstring "\>" \>\>% GreaterThan

let lt = pstring "\<" \>\>% LessThan

let op = spaces \>\>. choice[plus;minus;divide;mult;carrot;gt;lt]  
[/fsharp]

## Expressions

But what was great was parsing expressions. These were complicated because I had to avoid left recursion, and in my C# parser I had a lot of edge conditions and had to deal with special backtracking. It was a nightmare. With FParsec you can create forward recursive parsers, basically you create a dummy variable that you use as the recursive parser. Later you populate a tied reference to it with what are the available recursive parser implementations.

[fsharp]  
// create a forward reference  
// the expr is what we'll use in our parser combinators  
// the exprImpl we'll populate with all the recursive options later  
let expr, exprImpl = createParserForwardedToRef()

let expression1 = spaces \>\>? choice[floatNum;intNum;literal;variable]

let between a b p = pstring a \>\>. p .\>\> pstring b

let bracketExpression = expr |\> between "(" ")"

let lhExpression = choice[expression1; bracketExpression]

let expressionOperation = lhExpression \>\>=? fun operandL -\>  
 op \>\>=? fun operator -\>  
 choice[expr;bracketExpression] \>\>=? fun operandR -\>  
 preturn (operandL, operator, operandR) |\>\> Full

do exprImpl := spaces \>\>. choice[attempt expressionOperation;  
 attempt bracketExpression;  
 expression1]  
[/fsharp]

`expression1` is a type of expression that is just a single element. `expressionOperation` is an expression that has an operator inbetween two expressions. To avoid left recursion, the left hand side of an expression is limited to either expressions of single elements, or expressions encapsulated in parenthesis. Then the right hand side can be either a parenthesis expression, or a regular expression. You'll notice that `expr` isn't actually defined yet when its used here on line 16. It's a placeholder for the recursive parser, which is tied to the mutable reference cell (exprImpl) that I populate after I've defined all the parsers. This lets you define a parser that can actually call itself recursively! Neat.

The `>>=?` operator applies the result to the following function, but if it fails, backtracks to the beginning of that parsers state.

## Scope

I defined a scope as any valid statements between two curly brackets.

[fsharp]  
let funcInners, funcInnersImpl = createParserForwardedToRef()

let scope = parse{  
 do! spaces  
 do! skipStringCI "{"  
 do! spaces  
 let! text = opt funcInners  
 do! spaces  
 do! skipStringCI "}"  
 do! spaces  
 return Scope(text)  
}  
[/fsharp]

The forward parser here is going to be populated at the very end of my parser, since it will allow for any kind of statement like while loops, for loops, conditionals, assignment, etc. All the scope parser cares about is that it got stuff between curly brackets. Using this we can leverage it anywhere else that has curly brackets. Later on in the program I set this up:

[fsharp]  
(\* things that can be in functions \*)

do funcInnersImpl := many1 (spaces \>\>? choice [scope; func; statement])  
[/fsharp]

Which allows scopes, statements (delineated by semicolons), or other functions, to appear inside of a function or scope. So you can see how a scope and function can be recursive (by containing other scopes and functions inside of them).

Anyways, lets parse a function

[fsharp]  
let innerArgs = sepEndBy1 (expr |\>\> Element) (pstring ",")  
let arguments = innerArgs |\> between "(" ")"

let func = parse {  
 do! skipStringCI "func"  
 do! spaces  
 let! name = opt (many1Chars (satisfy isAsciiLetter))  
 let! arguments = opt arguments  
 do! spaces  
 do! skipStringCI "-\>"  
 let! scope = scope  
 return Function(name, arguments, scope)  
}  
[/fsharp]

## Conditionals

Conditionals were fun, because you can have an if statement, an if/else, or an if/elseif, or an if/elseif/.../else combo. In my previous C# parser I covered each type independently so I had a lot of extra overlap, but this time I wanted to see if I could create an aggregate parser combinator to handle all these scenarios for me in one.

[fsharp]  
let conditionalParser, conditionalParserImpl = createParserForwardedToRef()

let ifBlock = parse{  
 do! skipStringCI "if"  
 let! condition = expr |\> between "(" ")"  
 do! spaces  
 let! onTrue = scope  
 do! spaces

let elseKeyword = skipStringCI "else" .\>\> spaces

let elseParse = parse{  
 do! elseKeyword  
 let! onFalse = scope  
 return (condition, onTrue, Some(onFalse)) |\> Conditional  
 }

let elseIfParse = parse{  
 do! elseKeyword  
 let! onFalse = conditionalParser  
 return (condition, onTrue, Some(onFalse)) |\> Conditional  
 }

let noElseParse = parse{  
 return (condition, onTrue, None) |\> Conditional  
 }

let! result = choice[attempt elseIfParse;elseParse;noElseParse]  
 return result  
}  
[/fsharp]

This time, I created a recursive parser that optionally removes an else statement and captures the scope, or removes the else element and calls back into the if parser, or just terminates (so an if with no else). The final result is a 3 way alternative that the if block can evaluate.

## Loops

Here is a while loop

[fsharp]  
let whileLoop = (pstring "while" \>\>. spaces) \>\>. (expr |\> between "(" ")") \>\>= fun predicate -\>  
 scope \>\>= fun body -\>  
 preturn (WhileLoop(predicate, body))  
[/fsharp]

And here is a for loop

[fsharp]  
let assign = parse{  
 let! ex = expr  
 do! spaces  
 do! skipStringCI "="  
 do! spaces  
 let! assignEx = expr  
 do! spaces  
 return (ex, assignEx) |\> Assign  
}

let forLoop =  
 let startCondition = assign .\>\> pstring ";"  
 let predicate = expr .\>\> pstring ";"  
 let endCondition = expr  
 let forKeyword = pstring "for" .\>\> spaces

let forItems = tuple3 startCondition predicate endCondition |\> between "(" ")"

forKeyword \>\>. forItems .\>\>. scope \>\>= fun ((start, predicate, end), body) -\>  
 preturn (start, predicate, end, body) |\>\> ForLoop  
[/fsharp]

Which gives you a result like this

[code]  
test "for(x=1;y\<z;y+1){}";;  
val it : Ast list =  
 [ForLoop  
 (Assign (Variable "x",Float 1.0),  
 Full (Variable "y",LessThan,Variable "z"),  
 Full (Variable "y",Plus,Float 1.0),Scope null)]  
[/code]

## Conclusion

My fiance is probably pissed that I spent 4th of july working on parser combinators, but fparsec is just too fun not to. I really can't wait for an opportunity to use this for some production code, since I'm extremely happy with fparsecs abilities and the experience of working in it.

For the full parser check out this [fsharp snippet](http://fssnip.net/iJ).

