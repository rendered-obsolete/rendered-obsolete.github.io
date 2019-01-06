---
layout: post
title: Span C# 7.2
tags:
- csharp
- performance
---

Specification:
https://github.com/dotnet/corefxlab/blob/master/docs/specs/span.md
Practical MSDN article:
https://msdn.microsoft.com/en-us/magazine/mt814808.aspx
More:
https://blogs.msdn.microsoft.com/mazhou/2018/03/25/c-7-series-part-10-spant-and-universal-memory-management/


Medium post on span/pinvoke:
https://medium.com/@antao.almada/p-invoking-using-span-t-a398b86f95d3
https://adamsitnik.com/Span/

Fixed:
https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/fixed-statement
https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/unsafe-code-pointers/fixed-size-buffers

stackalloc:
https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/stackalloc

```
<PropertyGroup>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

Related JIT changes:
https://blogs.msdn.microsoft.com/dotnet/2017/10/16/ryujit-just-in-time-compiler-optimization-enhancements/

https://adamsitnik.com/Array-Pool/

SIMD support:
https://github.com/dotnet/corefx/issues/22940

http://mattwarren.org/2018/06/15/Tools-for-Exploring-.NET-Internals/
https://adamsitnik.com/Disassembly-Diagnoser/