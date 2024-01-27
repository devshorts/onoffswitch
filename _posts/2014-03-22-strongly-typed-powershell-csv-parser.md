---
layout: post
title: Strongly typed powershell csv parser
date: 2014-03-22 20:25:42.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- bytecode
- csv
- F#
- powershell
- reflection
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560310780;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4598;}i:1;a:1:{s:2:"id";i:4529;}i:2;a:1:{s:2:"id";i:4213;}}}}

permalink: "/2014/03/22/strongly-typed-powershell-csv-parser/"
---
Somehow I missed the powershell boat. I've been a .NET developer for years and I trudged through using the boring old cmd terminal, frequently mumbling about how much I missed zsh. But something snapped and I decided to really dive into powershell and learn why those who use it really love the hell out of it. After realizing that the reason everyone loves it is because everything is strongly typed and you can use .NET in your shell I was totally sold.

My first forays into powershell included customizing the shell environment. First I got [conemu](https://code.google.com/p/conemu-maximus5/) and made it look nice and pretty. Next was to get an ls highlighting module, since I love that about unix shells.

I set up a few fun aliases in [my profile](https://github.com/devshorts/powershell_scripts) and felt ready to conquer the world! My next experiment was to try and create an actual binary cmdlet. I figured, what better way than to create a csv reader. Now, I realize there is already an `Import-Csv` cmdlet that types your code, but I figured I'd write one from scratch, since apparently that's what I tend to do (instead of inventing anything new).

My hope was to make it so that it would emit strongly typed objects (which it does), but forwarning, you don't get intellisense on it in the shell. This is due to the fact that types are generated at runtime and not compile time.

For the lazy, here is a link to the [github](https://github.com/devshorts/Playground/tree/master/PowerShellCmdlets/Csv).

## The Plan

At first I thought I'd just wrap the F# csv type provider, but I realized that the type provider needs a template to generate its internal data classes. That won't do here because the cmdlet needs to accept any arbitrary csv file and strongly type at runtime.

To solve that, I figured I could leverage the [F# data csv library](http://fsharp.github.io/FSharp.Data/library/CsvFile.html) which would do the actual csv parsing, and then emit runtime bytecode to create data classes representing the header values.

As emitting bytecode is a pain in the ass, I wanted to keep my data classes simple. If I had a csv like:

```
  
Name,Age,Title  
Anton,30,Sr Engineer  
Faisal,30,Sr Engineer  

```

Then I wanted to emit a class like

```csharp
  
public class Whatever{  
 public String Name;  
 public String Age;  
 public String Title;

public Whatever(String name, String age, String title){  
 Name = name;  
 Age = age;  
 Title = title;  
 }  
}  

```

Since that would be the bare minimum that powershell would need to display the type.

## Emitting bytecode

First, lets look at the final result of what we need. The best way to do this is to create a sample type in an assembly and then to use `Ildasm` (an IL disassembler) to view the bytecode. For example, the following class

```csharp
  
using System;

namespace Sample  
{  
 public class Class1  
 {  
 public String foo;  
 public String bar;

public Class1(String f, String b)  
 {  
 foo = f;  
 bar = b;  
 }  
 }  
}  

```

Decompiles into this:

```
  
.method public hidebysig specialname rtspecialname  
 instance void .ctor(string f,  
 string b) cil managed  
{  
 // Code size 24 (0x18)  
 .maxstack 8  
 IL\_0000: ldarg.0  
 IL\_0001: call instance void [mscorlib]System.Object::.ctor()  
 IL\_0006: nop  
 IL\_0007: nop  
 IL\_0008: ldarg.0  
 IL\_0009: ldarg.1  
 IL\_000a: stfld string Sample.Class1::foo  
 IL\_000f: ldarg.0  
 IL\_0010: ldarg.2  
 IL\_0011: stfld string Sample.Class1::bar  
 IL\_0016: nop  
 IL\_0017: ret  
} // end of method Class1::.ctor  

```

While I didn't just divine how to write bytecode by looking at the IL (I followed some other blog posts), when I got an "invalid bytecode" CLR runtime error, it was nice to be able to compare what I was emitting which what I expected to emit. This way simple errors (like forgetting to load something on the stack) became pretty apparent.

To emit the proper bytecode, we need a few boilerplate items: an assembly, a type builder, an assembly builder, a module builder, and a field builder. These are responsible for the metadata you need to finally emit your built type.

```fsharp
  
let private assemblyName = new AssemblyName("Dynamics")

let private assemblyBuilder = AppDomain.CurrentDomain.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.RunAndSave)

let private moduleBuilder = assemblyBuilder.DefineDynamicModule(assemblyName.Name, assemblyName.Name + ".dll")

let private typeBuilder typeName = moduleBuilder.DefineType(typeName, TypeAttributes.Public)

let private fieldBuilder (typeBuilder:TypeBuilder) name fieldType : FieldBuilder =  
 typeBuilder.DefineField(name, fieldType, FieldAttributes.Public)

let private createConstructor (typeBuilder:TypeBuilder) typeList =  
 typeBuilder.DefineConstructor(MethodAttributes.Public, CallingConventions.Standard, typeList |\> List.toArray)


```

None of this is really all that interesting and hopefully is self explanatory.

The `fieldBuilder` is important since that will let us declare our local fields. In fact, once we've declared our local fields using the builder, the only bytecode we have to emit is the constructor (which accepts arguments and instantiates fields in them).

Here is the necessary code to build such a constructor.

```fsharp
  
let private callDefaultConstructor (gen: ILGenerator) =  
 let objType = typeof\<obj\>  
 gen.Emit(OpCodes.Call, objType.GetConstructor(Type.EmptyTypes))  
 gen.Emit(OpCodes.Ldarg\_0)

let private loadThis (gen: ILGenerator) =  
 gen.Emit(OpCodes.Ldarg\_0)  
 gen

let private emitNewInstanceRef (gen : ILGenerator) =  
 gen |\> loadThis |\> callDefaultConstructor

let private assignField (argIndex : int) (field : FieldBuilder) (gen : ILGenerator) =  
 gen.Emit(OpCodes.Ldarg, argIndex)  
 gen.Emit(OpCodes.Stfld, field)  
 gen

let private loadConstructorArg (gen : ILGenerator) ((num, field) : int \* FieldBuilder) =  
 gen |\> loadThis |\> assignField num field

let private completeConsructor (gen : ILGenerator) = gen.Emit(OpCodes.Ret)

let private build (fields : FieldBuilder list) (cons : ConstructorBuilder) =  
 let generator = cons.GetILGenerator()

generator |\> emitNewInstanceRef

let fieldsWithIndexes = fields |\> List.zip [1..(List.length fields)]

fieldsWithIndexes  
 |\> List.map (loadConstructorArg generator)  
 |\> ignore

generator |\> completeConsructor  

```

A few points of interest.

- Calls that make reference to OpCodes.Ldarp\_0 are loading the "this" object to work on. 
- OpCodes.Stdfld sets the passed in field to the value previously pushed on the stack.
- Opcodes.Ldarg with the index passed to it is a dynamic way of saying "load argument X onto the stack"

The final piece of the block is to tie it all together. Create field instances, take the target types and create a constructor, then return the type.

```fsharp
  
type FieldName = string  
type TypeName = string

let make (name : TypeName) (types : (FieldName \* Type) list)=  
 let typeBuilder = typeBuilder name  
 let fieldBuilder = fieldBuilder typeBuilder  
 let createConstructor = createConstructor typeBuilder  
 let fields = types |\> List.map (fun (name, ``type``) -\> fieldBuilder name ``type``)  
 let definedConstructor = types |\> List.map snd |\> createConstructor

definedConstructor |\> build fields

typeBuilder.CreateType()  

```

## Instantiating your type

Lets say we have a record that describes a field, its type, and a target value

```fsharp
  
type DynamicField = {  
 Name : String;  
 Type : Type;  
 Value: obj;  
}  

```

Then we can easily instantiate a target type with

```fsharp
  
let instantiate (typeName : TypeName) (objInfo : DynamicField list) =  
 let values = objInfo |\> List.map (fun i -\> i.Value) |\> List.toArray  
 let types = objInfo |\> List.map (fun i -\> (i.Name, i.Type))

let t = make typeName types

Activator.CreateInstance(t, values)  

```

It's important to note that `values` is an `obj []`. Because its an object array we can pass it to the activates overloaded function that wants a `params obj[]` and so it'll treat each object in the object array as another argument to the constructor.

## Dynamic static typing of CSV's

Since there is a way to dynamically create classes at runtime, it should be easy for us to leverage this to do the csv strong typing. In fact, the entire reader is this and emits to you a list of strongly typed entries:

```fsharp
  
open System  
open System.Reflection  
open System.IO  
open DataEmitter  
open FSharp.Data.Csv

module CsvReader =  
 let rand = System.Random()

let randomName() = rand.Next (0, 999999) |\> string

let defaultHeaders size = [0..size] |\> List.map (fun i -\> "Unknown Header " + (string i))

let load (stream : Stream) =  
 let csv = CsvFile.Load(stream).Cache()

let headers = match csv.Headers with  
 | Some(h) -\> h |\> Array.toList  
 | None -\> csv.NumberOfColumns |\> defaultHeaders

let fields = headers |\> List.map (fun fieldName -\> (fieldName, typeof\<string\>))

let typeData = make (randomName()) fields

[  
 for item in csv.Data do  
 let paramsArr = item.Columns |\> Array.map (fun i -\> i :\> obj)  
 yield Activator.CreateInstance(typeData, paramsArr)  
 ]  

```

The `randomName()` is a silly workaround to make sure I don't create the same `Type` in an assembly. Each time you run the csv reader it'll create a new random type representing that csv's data. I could maybe have optimized this that if someone calls in for a type with the same list of headers that another type had then to re-use that type instead of creating a duplicate, oh well.

## Using the reader from the cmdlet

Like I mentioned in the beginning, there is a major flaw here. The issue is that since my types are generated at runtime (which was really fun to do), it doesn't help me at all. Cmdlet's need to expose their output types via an `OutputType` attribute, and since its an attribute I can't expose the type dynamically.

Either way, here is the entire csv cmdlet

```fsharp
  
namespace CsvHandler

open DataEmitter  
open System.Management.Automation  
open System.Reflection  
open System  
open System.IO

[\<Cmdlet("Read", "Csv")\>]  
type CsvParser() =  
 inherit PSCmdlet()

[\<Parameter(Position = 0)\>]  
 member val File : string = null with get, set

override this.ProcessRecord() =  
 let (fileNames, \_) = this.GetResolvedProviderPathFromPSPath this.File

for file in fileNames do  
 use fileStream = File.OpenRead file

fileStream  
 |\> CsvReader.load  
 |\> List.toArray  
 |\> this.WriteObject  

```

This reads an implicit file name (or file with wildcards) and leverages the inherited `PsCmdlet` class to resolve the path from the passed in file (or expand any splat'd files like `some*`). All we do now is pass each file stream to the reader, convert to an array, and pass it to the next item in the powershell pipe.

## See it in action

Maybe this whole exercise was overkill, but let's finish it out anyways. Let's say we have a csv like this:

```
  
Year,Make,Model,Description,Price  
1997,Ford,E350,"ac, abs, moon",3000.00  
1999,Chevy,"Venture ""Extended Edition""","",4900.00  
1999,Chevy,"Venture ""Extended Edition, Very Large""",,5000.00  
1996,Jeep,Grand Cherokee,"MUST SELL!  
air, moon roof, loaded",4799.00  

```

We can do the following

![output1](http://onoffswitch.net/wp-content/uploads/2014/03/output1.png)

And filter on items

![filter](http://onoffswitch.net/wp-content/uploads/2014/03/filter.png)

## Cleanup

After getting draft one done, I thought about the handling of the IL generator in the Data Emitter. There are two things I wanted to accomplish:

1. Clean up having to seed the generator reference to all the functions  
2. Clean up passing an auto incremented index to the field initializer

After some mulling I realized that implementing a computation expression to handle the seeded state would be perfect for both scenarios. We can create an IlBuilder computation expression that will hold onto the reference of the generator and pass it to any function that uses `do!` syntax. We can do the same for the auto incremented index with a different builder. Let me show you the final result and then the builders:

```fsharp
  
let private build (fields : FieldBuilder list) (cons : ConstructorBuilder) =  
 let generator = cons.GetILGenerator()

let ilBuilder = new ILGenBuilder(generator)

let forNextIndex = new IncrementingCounterBuilder()

ilBuilder {  
 do! loadThis  
 do! callDefaultConstructor  
 do! loadThis

for field in fields do  
 do! loadThis  
 do! forNextIndex { return loadArgToStack }  
 do! field |\> setFieldFromStack

do! emitReturn  
 }  

```

And both builders:

```fsharp
  
(\* encapsulates an incrementable index \*)  
type IncrementingCounterBuilder () =  
 let mutable start = 0  
 member this.Return(expr) =  
 start \<- start + 1  
 expr start

(\* Handles automatically passing the il generator through the requested calls \*)  
type ILGenBuilder (gen: ILGenerator) =  
 member this.Bind(expr, func)=  
 expr gen  
 func () |\> ignore

member this.Return(v) = ()  
 member this.Zero () = ()  
 member this.For(col, func) = for item in col do func item  
 member this.Combine expr1 expr2 = ()  
 member this.Delay expr = expr()  

```

Now all mutability and state is contained in the expression. I think this is a much cleaner implementation and the functions I used in the builder workflow didn't have to have their function signatures changed!

## Conclusion

Sometimes you just jump in and don't realize the end goal won't work, but I did learn a whole lot figuring this out so the time wasn't wasted.

Check out full source at [my github](https://github.com/devshorts/Playground/tree/master/PowerShellCmdlets/Csv).

