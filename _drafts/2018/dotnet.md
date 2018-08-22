
## Converting libs

```
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(14,12,14,54): error CS0579: Duplicate 'System.Reflection.AssemblyCompanyAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(15,12,15,60): error CS0579: Duplicate 'System.Reflection.AssemblyConfigurationAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(16,12,16,58): error CS0579: Duplicate 'System.Reflection.AssemblyFileVersionAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(18,12,18,54): error CS0579: Duplicate 'System.Reflection.AssemblyProductAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(19,12,19,52): error CS0579: Duplicate 'System.Reflection.AssemblyTitleAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(20,12,20,54): error CS0579: Duplicate 'System.Reflection.AssemblyVersionAttribute' attribute
```

```xml
<PropertyGroup>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
</PropertyGroup>
```

## dotnet

I've previously used [MonoDevelop](https://www.monodevelop.com/) for some time.  Post Microsoft acquisition it was rebranded [Visual Studio for Mac](https://docs.microsoft.com/en-us/visualstudio/mac/) and there was much rejoicing.  The honeymoon is over:

```bash
mkdir l0core
cd l0core
# Create l0core.sln
dotnet new sln
# Create l0core/l0core.csproj
dotnet new console --name l0core
# Add l0core.csproj to l0core.sln
dotnet sln add l0core
# Add a bunch of existing projects


dotnet add l0core package NDesk.Options
```

One thing that was a surprised was although you could add an existing project to the sln with:
```bash
dotnet sln add ../layer0/Layer0Shared/
```
You couldn't easily add a project reference, for example:
```bash
dotnet add l0core reference Layer0Shared
```

Instead, you have to refer to the actual project:
```bash
dotnet add l0core reference ../layer0/Layer0Shared/
```

Reference `..\..\layer0\Layer0Shared\Layer0Shared.csproj` added to the project.

When I created the project [VS Code C# guide](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code#faq) mentions debugging and stuff that didn't work.  Down at the bottom, the [FAQ](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code#faq) mentions `.Net: Generate Assets for Build and Debug`.  Rebuffed with:
```
Could not locate .NET Core project.  Assets were not generated.
```

Seems to be [this bug](https://github.com/OmniSharp/omnisharp-vscode/issues/1425).  In any case, closing and restarting VS Code seemed to fix the issue.

__Start Debugging__ and immediately failing was great:  
![]({{ "/assets/vscode_debug.png" | absolute_url }})