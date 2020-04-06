---
layout: post
title: Determining 64bit or 32 bit .NET assemblies
date: 2013-05-16 16:28:26.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- 64 bit
- Utilities
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560340851;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3837;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4800;}}}}

permalink: "/2013/05/16/determining-64bit-32-bit-assemblies/"
---
I work on a 64 bit machine but frequently deploy to 32 bit machines. The code I work on though has native hooks so I always need to deploy assembly entry points at 32 bit. This means I am usually paranoid about the build configuration. However, sometimes things slip up and a 64 bit dll gets sent out or an entrypoint is built with `ANY CPU` set. Usually this is caught on our continuous build server with some cryptic reason for a unit test that should be working is actually failing.

When this happens, what you'll get is a message like this:

[code]  
Unhandled Exception: System.BadImageFormatException: Could not load file or assembly 'Some.dll' or one of its dependencies. An attempt was made to load a program with  
 an incorrect format.  
 at Test.Program.Run(Args args, Boolean fastStart)  
 at Test.ProgramMain(String[] args) in Program.cs:line 36  
[/code]

The first thing I do here is to try and figure out which of these dll's is built at the wrong type. The easiest way I've found to do that is to have a simple app that reflectively loads all the assemblies in a directory and tests their image format:

[csharp]  
class Program  
{  
 static void Main(string[] args)  
 {  
 if (args.Length != 1)  
 {  
 Console.WriteLine("Usage: \<directory to test for dlls\>");  
 return;  
 }

var dir = args[0];

Console.WriteLine();  
 Console.WriteLine("This machine is {0}", Is64BitOperatingSystem ? "64 bit" : "32 bit");  
 Console.WriteLine();

foreach (var file in Directory.EnumerateFiles(dir))  
 {  
 if (Path.GetExtension(file) == ".dll" || Path.GetExtension(file) == ".exe")  
 {  
 try  
 {  
 Assembly assembly = Assembly.ReflectionOnlyLoadFrom(file);  
 PortableExecutableKinds kinds;  
 ImageFileMachine imgFileMachine;  
 assembly.ManifestModule.GetPEKind(out kinds, out imgFileMachine);

Console.WriteLine("{0,-40} - {1,-15} - {2, -10}",  
 Path.GetFileName(file),  
 imgFileMachine,  
 kinds);  
 }  
 catch (Exception ex)  
 {  
 var err = "error";

if (ex.Message.Contains("The module was expected to contain an assembly manifest."))  
 {  
 err = "native";  
 }

Console.WriteLine("{0,-40} - {1,-15}", Path.GetFileName(file), err);  
 }  
 }  
 }  
 }

public static bool Is64BitOperatingSystem  
 {  
 get  
 {  
 // Clearly if this is a 64-bit process we must be on a 64-bit OS.  
 if (IntPtr.Size == 8)  
 return true;  
 // Ok, so we are a 32-bit process, but is the OS 64-bit?  
 // If we are running under Wow64 than the OS is 64-bit.  
 bool isWow64;  
 return ModuleContainsFunction("kernel32.dll", "IsWow64Process") && IsWow64Process(GetCurrentProcess(), out isWow64) && isWow64;  
 }  
 }

static bool ModuleContainsFunction(string moduleName, string methodName)  
 {  
 IntPtr hModule = GetModuleHandle(moduleName);  
 if (hModule != IntPtr.Zero)  
 return GetProcAddress(hModule, methodName) != IntPtr.Zero;  
 return false;  
 }

[DllImport("kernel32.dll", SetLastError = true)]  
 [return: MarshalAs(UnmanagedType.Bool)]  
 extern static bool IsWow64Process(IntPtr hProcess, [MarshalAs(UnmanagedType.Bool)] out bool isWow64);  
 [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]  
 extern static IntPtr GetCurrentProcess();  
 [DllImport("kernel32.dll", CharSet = CharSet.Auto)]  
 extern static IntPtr GetModuleHandle(string moduleName);  
 [DllImport("kernel32.dll", CharSet = CharSet.Ansi, SetLastError = true)]  
 extern static IntPtr GetProcAddress(IntPtr hModule, string methodName);  
}  
[/csharp]

Running the app will print out something like this:

[csharp highlight="16,14"]

This machine is 64 bit

7z.dll - native  
7z64.dll - native  
antlr.runtime.dll - I386 - ILOnly  
Local.Common.dll - I386 - ILOnly, Required32Bit  
BCrypt.Net.dll - I386 - ILOnly  
ICSharpCode.SharpZipLib.dll - I386 - ILOnly  
log4net.dll - I386 - ILOnly  
Lucene.Net.dll - I386 - ILOnly  
nunit.framework.dll - I386 - ILOnly  
pthreadVC2.dll - native  
SevenZipSharp.dll - I386 - ILOnly  
LocalInterop.dll - I386 - Required32Bit  
swscale-0.dll - native  
App.exe - I386 - ILOnly  
System.Reactive.dll - I386 - ILOnly  
XmlDiffPatch.dll - I386 - ILOnly  
[/csharp]

There you go, the application was built at `ANY CPU`. Anything marked with `ILOnly` can run on both 64bit and 32bit. If it is marked as only `Required32Bit` then it'll only work from a 32 bit process. Since I'm on a 64 bit machine running an ANY CPU process, the OS attempted to load the app at a 64 bit program format. This means all DLL's it loads have to support either ANY or 64 bit. Unfortunately, the interop dll is 32 bit only so that's what is causing the error.

If you're wondering how I knew that exception was for native code, the `native` determination is due to native dll's missing an [assembly manifest](http://msdn.microsoft.com/en-us/library/1w45z383(v=vs.100).aspx). [Only .NET files](http://stackoverflow.com/questions/12752828/are-manifest-files-only-for-managed-net-assemblies) contain an assembly manifest so I'm just testing that specific error.

