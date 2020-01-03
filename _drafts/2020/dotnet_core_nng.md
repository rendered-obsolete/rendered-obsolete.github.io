---
layout: post
title: .NET Core Primer using nng.NETCore
tags:
- nng
- dotnet
- csharp
- dotnetcore
---

[nngcat](https://nanomsg.github.io/nng/man/v1.1.0/nngcat.1)

```bash
# Create new console project called "nngcat"
dotnet new console --name nngcat
cd nngcat
dotnet add package Subor.nng.NETCore
# Build and run
dotnet build
dotnet run
```

In `nngcat.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Subor.nng.NETCore" Version="1.1.1" />
  </ItemGroup>

</Project>
```

Target is `netcoreapp3.0` because I'm using the .NET Core 3.0 preview.  If you use 2.2.2 it will default to `netcoreapp2.1`.

`Microsoft.Extensions.CommandLineUtils`

