---
layout: post
title: Building a custom lexer
date: 2013-02-25 16:54:43.000000000 -08:00
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
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561966458;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3565;}i:1;a:1:{s:2:"id";i:3016;}i:2;a:1:{s:2:"id";i:7777;}}}}

permalink: "/2013/02/25/building-a-custom-lexer/"
---
As a software engineer I spend all day (hopefully) writing code. I love code and I love that there are languages that help me solve problems and create solutions. But as an engineer I always want to know more about the tools I work with so I recently picked up "[Language Implementation Patterns](http://www.amazon.com/Language-Implementation-Patterns-Domain-Specific-Programming/dp/193435645X)" by [Terence Parr](http://www.cs.usfca.edu/~parrt/) and decided I was going to learn how to build a language. After reading through most of the book and working on examples for about 5 weeks I ended up building an interpreted [toy general purpose](https://github.com/devshorts/LanguageCreator) language that has features like:

- type inference
- partial functions
- static typing
- classes
- first class methods

The language I wrote is by no means production level. It's a toy, and it can do toy things. It can't do IO, it can't talk to the OS, it doesn't do GC, but it does evaluate expressions, call methods, basic exception handling, dynamic typing, and lets you do simple work.

Now that I'm done (but never finished) with the project I really appreciate how much goes into making a working language. The whole project increased my respect ten fold for people like [Don Syme](http://twitter.com/dsyme), [Tao Liu](https://twitter.com/ttliu2000), [Anders Hejlsberg](http://en.wikipedia.org/wiki/Anders_Hejlsberg), [Bjarne Stroustrup](http://en.wikipedia.org/wiki/Bjarne_Stroustrup), [Guido van Rossum](http://en.wikipedia.org/wiki/Guido_van_Rossum), and countless others who have made REAL languages. But now that I'm about done with my project, I wanted to share how parts of my language were done.

## Tokenizing

The first step to implementing a domain specific language, or even a general purpose language, is to figure out what the string representing the program means. You have to figure out that something like this:

```csharp
  
int foo = 1;

void func(){  
 print "abc";  
}  

```

Really means

```csharp
  
int  
foo  
equals  
number  
semicolon  
void  
word  
left parenthesis  
right parenthesis  
left bracket  
word  
quoted string  
semicolon  
right bracket  

```

To do this you need to first break up the program into a series of strings that you know are independent tokens, such as "int", and "foo". This is called tokenizing. Then you then need to go over that list and say "oh hey, I found this token 'int' but really it means `TokenType.Int`". Converting the strings to strong types is called lexing. Translating the string into strong types makes it significantly easier to work on it later.

While some grammars can be tokenized by breaking up on whitespace, for my language I found it easier to tokenize character by character. This means I don't have to worry about whitespace or special characters or any other kinds of weird delimiters. Granted, it's more resource intensive, but for a toy language that's OK. There are some great libraries out there that will let you define grammars for your language and will prebuild you lexers, tokenizers, parsers, tree walkers, etc (such as [ANTLR](http://www.antlr.org/)) but it can be fun doing this by hand the first time.

To tokenize my grammar I created a base class called `TokenizableStreamBase` that takes a function to generate a list of items and gives you some functionality to work on that stream. The base class lets me

- Consume an item. This means I've processed whatever is at the current index and we should move to the next item
- Snapshot the stream. I can keep track of where I am and if something happens later I can rollback to this point in the stream.
- Commit the stream. If I made a snapshot just discard it and continue from where I am now.

Let me give an example of why you want to use snapshots. Pretend you are trying to match the word "string", but the current tokenizer has "strint". Going character by character you would successfully match the word "strint" up until the last character. At that point you can tell it's not a "string" so you can roll back the stream to the start of the word and try something else.

Here is the base class:

```csharp
  
public class TokenizableStreamBase\<T\> where T : class  
{  
 public TokenizableStreamBase(Func\<List\<T\>\> extractor)  
 {  
 Index = 0;

Items = extractor();

SnapshotIndexes = new Stack\<int\>();  
 }

private List\<T\> Items { get; set; }

protected int Index { get; set; }

private Stack\<int\> SnapshotIndexes { get; set; }

public virtual T Current  
 {  
 get  
 {  
 if (EOF(0))  
 {  
 return null;  
 }

return Items[Index];  
 }  
 }

public void Consume()  
 {  
 Index++;  
 }

private Boolean EOF(int lookahead)  
 {  
 if (Index + lookahead \>= Items.Count)  
 {  
 return true;  
 }

return false;  
 }

public Boolean End()  
 {  
 return EOF(0);  
 }

public virtual T Peek(int lookahead)  
 {  
 if (EOF(lookahead))  
 {  
 return null;  
 }

return Items[Index + lookahead];  
 }

public void TakeSnapshot()  
 {  
 SnapshotIndexes.Push(Index);  
 }

public void RollbackSnapshot()  
 {  
 Index = SnapshotIndexes.Pop();  
 }

public void CommitSnapshot()  
 {  
 SnapshotIndexes.Pop();  
 }  
}  

```

And here is my entire tokenizer that creates a consumable stream of characters from the input source:

```csharp
  
public class Tokenizer : TokenizableStreamBase\<String\>  
{  
 public Lexer(String source) :  
 base(() =\> source.ToCharArray().Select(i =\> i.ToString(CultureInfo.InvariantCulture))  
 .ToList())  
 {  
 }  
}


```

In a later post I'll describe how I built my parser, which is responsible for creating abstract syntax trees and logical validation of the code. The parser re-uses the tokenizer stream base and instead of using characters (like the tokenizer stream) uses a stream of tokens.

## Lexing

A tokenizer is pretty useless though without creating meaningful tokens out of that source. To do that you need a way to match the characters to an expected literal. In my language I've created the concept of a matcher that can emit an object of type `Token`. Each matcher will be able to take the current tokenizer, consume characters until it finds what it wants, and if it found its match type, return the strongly typed token.

So, lets say I want to match the word "int" to token `Int` I'll have a matcher that knows it wants the string "int" and it will go through the lexer's current position matching while things are matching, or bailing if it encounters a character that it didn't expect. Being able to roll back a snapshot is called backtracking. Some grammars are clear enough to never have to do this. For example, if you will always have words that start with "i" and they are ALWAYS supposed to be "int" then you don't need to backtrack. If you fail to match then the syntax is invalid.

To build the matchers, I started with an abstract base:

```csharp
  
public abstract class MatcherBase : IMatcher  
{  
 public Token IsMatch(Tokenizer tokenizer)  
 {  
 if (tokenizer.End())  
 {  
 return new Token(TokenType.EOF);  
 }

tokenizer.TakeSnapshot();

var match = IsMatchImpl(tokenizer);

if (match == null)  
 {  
 tokenizer.RollbackSnapshot();  
 }  
 else  
 {  
 tokenizer.CommitSnapshot();  
 }

return match;  
 }

protected abstract Token IsMatchImpl(Tokenizer tokenizer);  
}  

```

This takes a `Tokenizer` and hands the tokenizer to whatever subclasses the abstract base for the match implementation. The base class will make sure to handle snapshots. If the matcher returns a non-null token then it will commit the snapshot otherwise it can roll it back and let the next matcher try.

As an example, here is the matcher for whitespace tokens. For whitespace I don't really care how much whitespace there was, just that there was whitespace. In fact in the end I discard whitespace completely since it doesn't really matter. If it found whitespace it'll return a new token of token type whitespace. Otherwise it'll return null.

```csharp
  
class MatchWhiteSpace : MatcherBase  
{  
 protected override Token IsMatchImpl(Tokenizer tokenizer)  
 {  
 bool foundWhiteSpace = false;

while (!tokenizer.End() && String.IsNullOrWhiteSpace(tokenizer.Current))  
 {  
 foundWhiteSpace = true;

tokenizer.Consume();  
 }

if (foundWhiteSpace)  
 {  
 return new Token(TokenType.WhiteSpace);  
 }

return null;  
 }  
}  

```

Below is another matcher that finds quoted strings. You can set the quote delimiter to be either a " or a '. So "this" matches and so does 'this'.

```csharp
  
public class MatchString : MatcherBase  
{  
 public const string QUOTE = "\"";

public const string TIC = "'";

private String StringDelim { get; set; }

public MatchString(String delim)  
 {  
 StringDelim = delim;  
 }

protected override Token IsMatchImpl(Tokenizer tokenizer)  
 {  
 var str = new StringBuilder();

if (tokenizer.Current == StringDelim)  
 {  
 tokenizer.Consume();

while (!tokenizer.End() && tokenizer.Current != StringDelim)  
 {  
 str.Append(tokenizer.Current);  
 tokenizer.Consume();  
 }

if (tokenizer.Current == StringDelim)  
 {  
 tokenizer.Consume();  
 }  
 }

if (str.Length \> 0)  
 {  
 return new Token(TokenType.QuotedString, str.ToString());  
 }

return null;  
 }  
}  

```

The next matcher (below) does the bulk of the work. This one finds built in keywords and special characters by taking an input string, and the final token it should represent. It determines if the current stream contains that token and then emits it. The idea is if you know "int" is a built in type, and it should match to some `TokenType.Int`, you can pass that info to a `MatchKeyword` instance and it'll find the token for you if it exists. The `Match` property contains the raw string you want to match on, and the `TokenType` property represents the strongly typed type that should be paired to the raw string:

```csharp
  
public class MatchKeyword : MatcherBase  
{  
 public string Match { get; set; }

private TokenType TokenType { get; set; }

/// \<summary\>  
 /// If true then matching on { in a string like "{test" will match the first cahracter  
 /// because it is not space delimited. If false it must be space or special character delimited  
 /// \</summary\>  
 public Boolean AllowAsSubString { get; set; }

public List\<MatchKeyword\> SpecialCharacters { get; set; }

public MatchKeyword(TokenType type, String match)  
 {  
 Match = match;  
 TokenType = type;  
 AllowAsSubString = true;  
 }

protected override Token IsMatchImpl(Tokenizer tokenizer)  
 {  
 foreach (var character in Match)  
 {  
 if (tokenizer.Current == character.ToString(CultureInfo.InvariantCulture))  
 {  
 tokenizer.Consume();  
 }  
 else  
 {  
 return null;  
 }  
 }

bool found;

if (!AllowAsSubString)  
 {  
 var next = tokenizer.Current;

found = String.IsNullOrWhiteSpace(next) || SpecialCharacters.Any(character =\> character.Match == next);  
 }  
 else  
 {  
 found = true;  
 }

if (found)  
 {  
 return new Token(TokenType, Match);  
 }

return null;  
 }  
}  

```

The special characters list is an injected list of keyword matchers that let the current matcher know when things are delimited. For example, we want to support both

```csharp
  
if(  

```

and

```csharp
  
if (  

```

In the first block, the keyword "if" is delimited by a special character "(" and not just whitespace. By using special characters AND whitespace as delimiters we can have whitespace agnostic code.

For my language I supported the following special characters and keywords:

```csharp
  
public enum TokenType  
{  
 Infer,  
 Void,  
 WhiteSpace,  
 LBracket,  
 RBracket,  
 Plus,  
 Minus,  
 Equals,  
 HashTag,  
 QuotedString,  
 Word,  
 Comma,  
 OpenParenth,  
 CloseParenth,  
 Asterix,  
 Slash,  
 Carat,  
 DeRef,  
 Ampersand,  
 Fun,  
 GreaterThan,  
 LessThan,  
 SemiColon,  
 If,  
 Return,  
 While,  
 Else,  
 ScopeStart,  
 EOF,  
 For,  
 Float,  
 Print,  
 Dot,  
 True,  
 False,  
 Boolean,  
 Or,  
 Int,  
 Double,  
 String,  
 Method,  
 Class,  
 New,  
 Compare,  
 Nil,  
 NotCompare,  
 Try,  
 Catch  
}  

```

It's a lot, but at the same time it's not nearly enough! Still, having a single matcher that can match on known tokens means extending the language at this level is quite easy. Here is the construction of the list of my matchers. Order here matters since it determines precedence. You can see now that to add new keywords or characters to the grammar only requires defining a new enum and updating the appropriate match list.

```csharp
  
private List\<IMatcher\> InitializeMatchList()  
{  
 // the order here matters because it defines token precedence

var matchers = new List\<IMatcher\>(64);

var keywordmatchers = new List\<IMatcher\>  
 {  
 new MatchKeyword(TokenType.Void, "void"),  
 new MatchKeyword(TokenType.Int, "int"),  
 new MatchKeyword(TokenType.Fun, "fun"),  
 new MatchKeyword(TokenType.If, "if"),  
 new MatchKeyword(TokenType.Infer, "var"),  
 new MatchKeyword(TokenType.Else, "else"),  
 new MatchKeyword(TokenType.While, "while"),  
 new MatchKeyword(TokenType.For, "for"),  
 new MatchKeyword(TokenType.Return, "return"),  
 new MatchKeyword(TokenType.Print, "print"),  
 new MatchKeyword(TokenType.True, "true"),  
 new MatchKeyword(TokenType.False, "false"),  
 new MatchKeyword(TokenType.Boolean, "bool"),  
 new MatchKeyword(TokenType.String, "string"),  
 new MatchKeyword(TokenType.Method, "method"),  
 new MatchKeyword(TokenType.Class, "class"),  
 new MatchKeyword(TokenType.New, "new"),  
 new MatchKeyword(TokenType.Nil, "nil")  
 };

var specialCharacters = new List\<IMatcher\>  
 {  
 new MatchKeyword(TokenType.DeRef, "-\>"),  
 new MatchKeyword(TokenType.LBracket, "{"),  
 new MatchKeyword(TokenType.RBracket, "}"),  
 new MatchKeyword(TokenType.Plus, "+"),  
 new MatchKeyword(TokenType.Minus, "-"),  
 new MatchKeyword(TokenType.Equals, "="),  
 new MatchKeyword(TokenType.HashTag, "#"),  
 new MatchKeyword(TokenType.Comma, ","),  
 new MatchKeyword(TokenType.OpenParenth, "("),  
 new MatchKeyword(TokenType.CloseParenth, ")"),  
 new MatchKeyword(TokenType.Asterix, "\*"),  
 new MatchKeyword(TokenType.Slash, "/"),  
 new MatchKeyword(TokenType.Carat, "^"),  
 new MatchKeyword(TokenType.Ampersand, "&"),  
 new MatchKeyword(TokenType.GreaterThan, "\>"),  
 new MatchKeyword(TokenType.LessThan, "\<"),  
 new MatchKeyword(TokenType.Or, "||"),  
 new MatchKeyword(TokenType.SemiColon, ";"),  
 new MatchKeyword(TokenType.Dot, "."),  
 };

// give each keyword the list of possible delimiters and not allow them to be  
 // substrings of other words, i.e. token fun should not be found in string "function"  
 keywordmatchers.ForEach(keyword =\>  
 {  
 var current = (keyword as MatchKeyword);  
 current.AllowAsSubString = false;  
 current.SpecialCharacters = specialCharacters.Select(i =\> i as MatchKeyword).ToList();  
 });

matchers.Add(new MatchString(MatchString.QUOTE));  
 matchers.Add(new MatchString(MatchString.TIC));

matchers.AddRange(specialCharacters);  
 matchers.AddRange(keywordmatchers);

matchers.AddRange(new List\<IMatcher\>  
 {  
 new MatchWhiteSpace(),  
 new MatchNumber(),  
 new MatchWord(specialCharacters)  
 });

return matchers;  
}  

```

To actually run through and get the tokens we do this

```csharp
  
public IEnumerable\<Token\> Lex()  
{  
 Matchers = InitializeMatchList();

var current = Next();

while (current != null && current.TokenType != TokenType.EOF)  
 {  
 // skip whitespace  
 if (current.TokenType != TokenType.WhiteSpace)  
 {  
 yield return current;  
 }

current = Next();  
 }  
}

.... define the match list ...

private Token Next()  
{  
 if (Lexer.End())  
 {  
 return new Token(TokenType.EOF);  
 }

return  
 (from match in Matchers  
 let token = match.IsMatch(Tokenizer)  
 where token != null  
 select token).FirstOrDefault();  
}  

```

And the only thing left is in the constructor of the Tokenizer

```csharp
  
public Lexer(String source)  
{  
 Tokenizer = new Tokenizer(source);  
}  

```

## Testing

Lets see it in action in a unit test. Keep in mind the tokenizer and lexer do only the most basic syntax validation, but not much. It's more about creating a typed token stream representing your code. Later we can use the typed token stream to create meaningful data structures representing the code.

```csharp
  
[Test]  
public void TestTokenizer()  
{  
 var test = @"function void int ""void int"" {} -\>\*/test^void,5,6,7 8.0";

var tokens = new Lexer(test).Lex().ToList();

foreach (var token in tokens)  
 {  
 Console.WriteLine(token.TokenType + " - " + token.TokenValue);  
 }  
}  

```

And this prints us out

```csharp
  
Word - function  
Void - void  
Int - int  
QuotedString - void int  
LBracket - {  
RBracket - }  
DeRef - -\>  
Asterix - \*  
Slash - /  
Word - test  
Carat - ^  
Void - void  
Comma - ,  
Int - 5  
Comma - ,  
Int - 6  
Comma - ,  
Int - 7  
Float - 8.0  

```

Now we're in a position that we can start parsing our language.

