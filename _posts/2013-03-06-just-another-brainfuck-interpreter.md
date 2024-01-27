---
layout: post
title: Just another brainfuck interpreter
date: 2013-03-06 00:36:30.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- brainfuck
- interpreter
- parser
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _wp_old_slug: a-brainfuck-interpreter
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560277132;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2735;}i:1;a:1:{s:2:"id";i:6;}i:2;a:1:{s:2:"id";i:3016;}}}}

permalink: "/2013/03/06/just-another-brainfuck-interpreter/"
---
<h2>Why?</h2>
<p>Honestly, why not? </p>
<h2>The entry point</h2>
<p>Not much to tell:</p>
<p>```csharp
<br />
static void Main(string[] args)<br />
{<br />
    var parser = new Parser(&quot;++++++++++[&gt;+++++++&gt;++++++++++&gt;+++&gt;+&lt;&lt;&lt;&lt;-]&gt;++.&gt;+.+++++++..+++.&gt;++.&lt;&lt;+++++++++++++++.&gt;.+++.------.--------
.\>+.\>.");

var instructions = parser.Instructions;

var interpreter = new Interpreter(instructions);

interpreter.Interpret();  
}  

```

## The container classes

Some data classes and enums:

```csharp
  
public enum Tokens  
 {  
 MoveFwd,  
 MoveBack,  
 Incr,  
 Decr,  
 While,  
 Print,  
 Input,  
 WhileEnd,  
 WhileStart,  
 Unknown  
 }

public class Instruction  
 {  
 public Tokens Token { get; set; }

public override string ToString()  
 {  
 return Token.ToString();  
 }  
 }

class While : Instruction  
 {  
 public While()  
 {  
 Token = Tokens.While;  
 }

public List\<Instruction\> Instructions { get; set; }  
 }  

```

## A helper function

A function to translate a character token into a known token

```csharp
  
private Tokens GetToken(char input)  
{  
 switch (input)  
 {  
 case '+':  
 return Tokens.Incr;  
 case '-':  
 return Tokens.Decr;;  
 case '\<':  
 return Tokens.MoveBack;  
 case '\>':  
 return Tokens.MoveFwd;  
 case '.':  
 return Tokens.Print;  
 case ',':  
 return Tokens.Input;  
 case '[':  
 return Tokens.WhileStart;  
 case ']':  
 return Tokens.WhileEnd;  
 }  
 return Tokens.Unknown;  
}  

```

## The parser

And the entire parser:

```csharp
  
public List\<Instruction\> Instructions { get; private set; }

public Parser(string source)  
{  
 Instructions = Tokenize(source.Select(GetToken)  
 .Where(token =\> token != Tokens.Unknown)  
 .ToList()).ToList();  
}

IEnumerable\<Instruction\> Tokenize(IEnumerable\<Tokens\> input)  
{  
 var stack = new Stack\<While\>();

foreach (var t in input)  
 {  
 switch (t)  
 {  
 case Tokens.WhileStart:  
 stack.Push(new While {Instructions = new List\<Instruction\>()});  
 break;  
 case Tokens.WhileEnd:  
 if (stack.Count == 0)  
 {  
 throw new Exception("Found a ] without a matching [");  
 }  
 if (stack.Count \> 1)  
 {  
 var top = stack.Pop();  
 stack.Peek().Instructions.Add(top);  
 }  
 else  
 {  
 yield return stack.Pop();  
 }  
 break;  
 default:  
 var instruction = new Instruction {Token = t};  
 if (stack.Count \> 0)  
 {  
 stack.Peek().Instructions.Add(instruction);  
 }  
 else  
 {  
 yield return instruction;  
 }  
 break;  
 }  
 }

if (stack.Count \> 0)  
 {  
 throw new Exception("Unmatched [found. Expecting]");  
 }  
}  

```

I took a different approach to parsing this time than usual. I didn't feel like having to deal with a consumable stream, so I linearly went through the token source. Each time I encountered a while loop start I pushed it onto the while loop stack. Anytime I had instructions, if I had stuff in the stack, I added it to the instruction list at the top of stack. As I hit while loop end tokens (]), I popped off the stack and yielded the aggregate instruction.

## The interpreter

The interpreter is dirt simple too. We have a 30,000 byte array of memory (per spec from wikipedia). The rest I think is self explanatory:

```csharp
  
public class Interpreter  
{  
 private readonly byte[] \_space = new byte[30000];

private int \_dataPointer;

private List\<Instruction\> Instructions { get; set; }

public Interpreter (List\<Instruction\> instructions)  
 {  
 Instructions = instructions;  
 }

public void Interpret()  
 {  
 InterpretImpl(Instructions);  
 }

private void InterpretImpl(IEnumerable\<Instruction\> instructions)  
 {  
 foreach(var instruction in instructions){  
 switch (instruction.Token)  
 {  
 case Tokens.Input:  
 \_space[\_dataPointer] = Convert.ToByte(Console.Read());  
 break;  
 case Tokens.Incr:  
 \_space[\_dataPointer]++;  
 break;  
 case Tokens.Decr:  
 \_space[\_dataPointer]--;  
 break;  
 case Tokens.Print:  
 Console.Write(Encoding.ASCII.GetString(new[] { \_space[\_dataPointer] }));  
 break;  
 case Tokens.MoveFwd:  
 \_dataPointer++;  
 break;  
 case Tokens.MoveBack:  
 \_dataPointer--;  
 break;  
 case Tokens.While:  
 while (\_space[\_dataPointer] != 0)  
 {  
 InterpretImpl((instruction as While).Instructions);  
 }  
 break;  
 }  
 }  
 }  
}  

```

## The source

Read the [source](https://github.com/devshorts/BrainFuckSharp), Luke

