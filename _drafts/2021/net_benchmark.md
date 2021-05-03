---
layout: post
title: Benchmarking in .NET
tags:
- dotnet
- csharp
- performance
- optimization
---

[BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)

https://benchmarkdotnet.org/articles/overview.html

```powershell
# Install benchmark template
dotnet new -i BenchmarkDotNet.Templates
# Create new benchmark project `benchmarks`
dotnet new benchmark -o benchmarks --console-app -v 0.12.1
# Add a project reference to `nng.NET.Shared`
dotnet add benchmarks reference ./nng.NET.Shared/nng.NET.Shared.csproj
```

- [BenchmarkDotNet template options](https://benchmarkdotnet.org/articles/guides/dotnet-new-templates.html)

Confusingly, you'll find references to installing `BenchmarkDotNet.Tool` global tool which has been deprecated (see issue [1670](https://github.com/dotnet/BenchmarkDotNet/issues/1670) and PR [1572](https://github.com/dotnet/BenchmarkDotNet/pull/1572)).  Likewise, the dotnet project template `BenchmarkDotNet.Templates` defaults to: NET Core App 3.0 (which is [out of support](https://aka.ms/dotnet-core-support)), and version 0.12.0 of BenchmarkDotNet (which doesn't support NET 5.0 and fails with `System.NotSupportedException: Unknown .NET Runtime`).

Open `benchmarks/benchmarks.csproj` and replace:
```diff
-<TargetFramework>netcoreapp3.0</TargetFramework>
+<TargetFramework>net5.0</TargetFramework>
```

In `benchmarks/Program.cs`, replace the main body with the following to get the same functionality as the benchmark runner CLI (from [the docs](https://benchmarkdotnet.org/articles/guides/console-args.html)):

```csharp
class Program
{
    static void Main(string[] args) 
        => BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);
}
```

Try running the benchmarks:
```powershell
dotnet run --project ./Benchmarks/benchmarks.csproj -- -f "benchmarks.*"
```

If you leave off `--filter` it will wait for input from stdin.


