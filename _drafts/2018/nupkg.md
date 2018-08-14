
Recently started looking at publishing [our C# libraries to nuget.org](https://www.nuget.org/profiles/subor).

[Subor.NNanomsg.NETStandard](https://www.nuget.org/packages/Subor.NNanomsg.NETStandard/) ([our fork](https://github.com/zplus/NNanomsg) of [NNanomsg](https://github.com/mhowlett/NNanomsg)- a C# library for [nanomsg](https://nanomsg.org/)) was particularly frustrating because it involves a native library `nanomsg.dll`.

## References

Google un-earthed several promising resources:

- Stackoverflow [#1](https://stackoverflow.com/questions/19478775/add-native-files-from-nuget-package-to-project-output-directory) and [#2](https://stackoverflow.com/questions/40104838/automatic-native-and-managed-dlls-extracting-from-nuget-package)
- [MS docs on using nuget to create native packages](https://docs.microsoft.com/en-us/nuget/create-packages/native-packages)
- [Blog by former SignalR/ASP.NET guy](https://blog.3d-logic.com/2015/11/10/using-native-libraries-in-asp-net-5/)

But I struggled making a package that worked; `nanomsg.dll` wasn't in the output directory.

## Tips

- The `.targets` file mentioned in several write-ups seems to no longer be necessary (VS 15.7.4 and nuget.exe 4.7.0) unless your package will be consumed by c++ projects.

- Your managed libraries need to get packaged in `/lib/TFM/`.  See
[Supporting multiple .NET framework versions](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks) and about [Target Framework Monikers (TFM)](https://docs.microsoft.com/en-us/nuget/reference/target-frameworks#supported-frameworks)

- Your native libraries need to get packaged in `/runtimes/RID/native/`.  See [Runtime ID Catalog](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog) not only has a complete list of all Runtime IDs (RIDs), but also gives a pretty detailed explanation.



- Familiarize yourself with nuget.exe
    - `nuget pack -Properties Configuration=Release` will build a release nupkg (default is debug) for the `.csproj` in the current directory.  The output assembly will automatically be included.

- Nuspec
    - A nuspec file with the same name as the csproj will automatically get included.  For example, `NNanomsg.NETStandard.csproj` and `NNanomsg.NETStandard.nuspec`.
    - [Replacement tokens](https://docs.microsoft.com/en-us/nuget/reference/nuspec#replacement-tokens) usable in nuspec
    - Using a nuspec together with a csproj is a bit clumsy
        - There's only replacement tokens for some meta-data values
        - Omitting a tag from the nuspec results in the value being undefined- even if it is set in the csproj
        - But nuspec seems to be the only way to include `/runtimes/` files in the package.  `<Content PackagePath="runtimes\xxxx" Pack="true" ... />` places things in a `content/` subfolder, `<None>` doesn't include them at all.


- It's less convenient, but you can directly pack the nuspec (e.g. `nuget pack xxx.nuspec`)
    - If you see the following warning:
        ```
        WARNING: NU5100: The assembly 'bin\Release\netstandard2.0\NNanomsg.NETStandard.dll' is not inside the 'lib' folder and hence it won't be added as a reference when the package is installed into a project. Move it into the 'lib' folder if it needs to be referenced.
        ```
        You likely need the following in your nuspec:
        ```
        <files>
            <file src="bin\$configuration$\netstandard2.0\*.dll" target="lib\netstandard2.0\"/>
            <file src="runtimes\**" target="runtimes/"/>
        </files>
        ```
    - If you get the following error:
        ```
        Attempting to build package from 'NNanomsg.NETStandard.nuspec'.
        Value cannot be null or an empty string.
        Parameter name: value
        ```
        The nuspec contains a replacement token (i.e. `$xxx$`) that needs to be replaced with an actual value


- The [NuGetPackageExplorer](https://github.com/NuGetPackageExplorer/NuGetPackageExplorer) gives a nice visual look at your nupkg and its meta-data.  Just make sure to __File->Close__ otherwise nuget.exe and Visual Studio will have problems accessing your nupkg.

- Installing the package doesn't cause the native binaries to get copied, you have to build the project.

- Assuming your nupkg contains 32-bit and 64-bit native libraries, `runtimes/win-x86/native/nanomsg.dll` and `runtimes/win-x64/native/nanomsg.dll`, respectively.  If _Platform target_ (project __Properties->Build__) of the consuming project is `Any CPU` __neither__ native assembly will be copied to the output directory.  You __must__ select either `x86` or `x64`:
![]({{ "/assets/vs_options_package_source.jpg" | absolute_url }})

- In Visual Studio, __Tools->Options->NuGet Package Manager->Package Sources__ add local directory to use as source for nuget packages:
![]({{ "/assets/vs_project_properties_platform_target.jpg" | absolute_url }})

- Visual Studio Package Manager (__View->Other Windows->Package Manager Console__) makes it easy to reload a local nupkg.  Make sure __Default project__ is set to the correct package and run, for example: `Install-Package D:\dev\NNanomsg\NNanomsg.NETStandard\NNanomsg.NETStandard.0.5.2.nupkg`.

