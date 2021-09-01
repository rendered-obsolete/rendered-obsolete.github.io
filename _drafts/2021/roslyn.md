---
layout: post
title: Generating C# Interop with Roslyn
tags:
- dotnet
- csharp
- roslyn
---

> C source > C# source

To generate C# source, the best choice is Roslyn.  But, how do we deal with the C?  There's a few solutions:

- Use a parser written with something like [ANTLR](https://www.antlr.org/) (_yikes_)
- Use [ClangSharp](https://github.com/microsoft/ClangSharp) to parse C source into an AST
- Add an option to ClangSharp to generate function pointers
- Use ClangSharp to generate `DllImport` bindings and then parse that with Roslyn

The last feels like the easiest because we won't have to wrangle a complex intermediary representation like clang/llvm AST; it's just a C#-to-C# translation with Roslyn.  We lose the flexibility and detail of the full AST, but it should be simpler and will be useful to projects that already use `DllImport`.

Adding a feature to ClangSharp is the nobler option, but I'm more interested in working with Roslyn and not having to deal with clang.  Maybe after I've got Roslyn figured out.  Plus, we need the `DllImport` bindings anyway because it looks like it's necessary to support pre-.NET 5.0/C# 9.0:
https://github.com/dotnet/runtime/discussions/40304

So the processing will be like:

> C source > ClangSharp > C# source w/ DllImport > Roslyn > C# source w/ function pointers




```
dotnet tool install --global ClangSharpPInvokeGenerator --version 11.0.0-beta3
```

Check [nuget](https://www.nuget.org/packages?q=ClangSharpPInvokeGenerator) for the latest version.

```
System.DllNotFoundException: Unable to load shared library 'libclang' or one of its dependencies.
```

Use a response file:
```
```

```powershell
# PowerShell needs the quotes, other shells may not
ClangSharpPInvokeGenerator '@clangsharp.rsp'
```

https://www.cazzulino.com/source-generators.html

https://docs.microsoft.com/en-us/archive/msdn-magazine/2017/may/net-core-cross-platform-code-generation-with-roslyn-and-net-core

