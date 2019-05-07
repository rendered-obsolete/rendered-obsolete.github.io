---
layout: post
title: C# 8.0 Switch and .NET 5.0
tags:
- dotnet
- csharp
---

With [Visual Studio 2019 released](https://devblogs.microsoft.com/visualstudio/visual-studio-2019-code-faster-work-smarter-create-the-future/), 
we're counting down to [.NET Core 3.0 and C# 8.0]({% post_url /2018/2018-12-05-netcore-3-csharp-8-preview %}) this fall.  Microsoft just announced the plan going forward: [.NET 5](https://devblogs.microsoft.com/dotnet/introducing-net-5/) (coverage on [VentureBeat](https://venturebeat.com/2019/05/06/microsoft-visual-studio-online-net-5-ml-net-1-0/) amongst others).

## .NET 5

Unless you're a long-time enthusiast of .NET, the entire ecosystem of ".NET Framework"/".NET Core"/"Mono", various languages (C#/F#/etc.), and associated [TLA](https://en.wikipedia.org/wiki/Three-letter_acronym)-soup of libraries is subtle and confusing.  Steering things in the same direction and unifying various sundry technologies is a goal for Fall 2020:

- Unify .NET Framework, .NET Core, and Xamarin/Mono into single platform.
    - .NET Framework and .NET Core become single open-source framework/runtime: ".NET 5".
    - Mono and CoreCLR (.NET Core runtime) as alternative "runtime experiences".
- Java interoperability on all platforms, Objective-C and Swift on some platforms.
- Improve [AOT compilation](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) story.
    - Full AOT needed for iOS, [WASM/WebAssembly](https://webassembly.org/), and some other platforms.
    - Partial AOT (to reduce startup time and memory usage) combined with [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation) (to maximize code support for generics, reflection, etc.) useful.
    - [Mono has had AOT](https://www.mono-project.com/docs/advanced/aot/) for a while, UWP apps have [.NET Native](https://docs.microsoft.com/en-us/dotnet/framework/net-native/), .NET Framework has older [NGEN tool](https://docs.microsoft.com/en-us/dotnet/framework/tools/ngen-exe-native-image-generator).

## C# 8.0 `switch`

We're [(most) excited about "nullable reference types"]({% post_url /2018/2018-12-05-netcore-3-csharp-8-preview %}) (to catch some `null`-related runtime exceptions at compile-time), but there's other good stuff coming in C# 8.  The humble `switch` was upgraded with [pattern matching]({% post_url /2019/2019-01-06-thrift_012 %}#unions) in C# 7 and gets a few new abilities in C# 8.

Here's the starting struct from the [.NET Blog post by Mads](https://devblogs.microsoft.com/dotnet/do-more-with-patterns-in-c-8-0/):
```csharp
class Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
    public void Deconstruct(out int x, out int y) => (x, y) = (X, Y);
}
```

Switch expressions allow `switch` to "return" a value:
```csharp
// Before (switch as a statement)
string Display(object o)
{
    switch (o)
    {
        case Point p:
            return $"({p.X}, {p.Y})";
        default:
            return "unknown";
    }
}
// After (switch as an expression)
string Display(object o)
{
    return o switch
    {
        Point p => $"({p.X}, {p.Y})",
        _       => "unknown"
    };
}
```

If the end of a switch expression is reached without making a match then an expression is thrown.

__Property patterns__ allow you to further refine matches based on accessible fields:
```csharp
// Before (switch as a statement)
string Display(object o)
{
    switch (o)
    {
        case Point p when p.X == 0 && p.Y == 0:
            return "origin";
        //...
    }
}
// After (switch as an expression)
string Display(object o)
{
    return o switch
    {
        Point { X: 0, Y: 0 } /* p */         => "origin",
        Point { X: var x, Y: var y } /* p */ => $"({x}, {y})",
        {}                                   => o.ToString(), // `o` is non-null and not a `Point`
        null => "null"
    };
}
```

__Positional patterns__ work with tuples and types- like `Point`- that have a `Deconstruct()` method (see [deconstructors](https://docs.microsoft.com/en-us/dotnet/csharp/deconstruct)):
```csharp
string Display(object o)
{
    return o switch
    {
        Point(0, 0)         => "origin",
        Point(var x, var y) => $"({x}, {y})",
        //...
    }
}

// General case for tuples
string DisplayXY(int x, int y, bool isValid)
    (x, y, isValid) switch {
        (_, _, false) => "invalid",
        (0, 0, true) => "origin",
        //...
    }
```

Overall, I'm not thrilled about `XYZ switch` syntax instead of `switch XYZ`, but these are otherwise welcome additions (especially coming from other languages with a similar feature).

For a longer write up also see this [recent MSDN magazine article](https://msdn.microsoft.com/en-us/magazine/mt833440.aspx?f=255&MSPPError=-2147217396).
