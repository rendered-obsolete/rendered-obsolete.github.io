`dotnet pack --help`:
```
Usage: dotnet pack [options] <PROJECT | SOLUTION>

Arguments:
  <PROJECT | SOLUTION>   The project or solution file to operate on. If a file is not specified, the command will search the current directory for one.

Options:
  -h, --help                            Show command line help.
  -o, --output <OUTPUT_DIR>             The output directory to place built packages in.
  --no-build                            Do not build the project before packing. Implies --no-restore.
  --include-symbols                     Include packages with symbols in addition to regular packages in output directory.
  --include-source                      Include PDBs and source files. Source files go into the 'src' folder in the resulting nuget package.
  -c, --configuration <CONFIGURATION>   The configuration to use for building the package. The default for most projects is 'Debug'.
  --version-suffix <VERSION_SUFFIX>     Set the value of the $(VersionSuffix) property to use when building the project.
  -s, --serviceable                     Set the serviceable flag in the package. See https://aka.ms/nupkgservicing for more information.
  /nologo, --nologo                     Do not display the startup banner or the copyright message.
  --interactive                         Allows the command to stop and wait for user input or action (for example to complete authentication).
  --no-restore                          Do not restore the project before building.
  -v, --verbosity <LEVEL>               Set the MSBuild verbosity level. Allowed values are q[uiet], m[inimal], n[ormal], d[etailed], and diag[nostic].
  --runtime <RUNTIME_IDENTIFIER>        The target runtime to restore packages for.
  --no-dependencies                     Do not restore project-to-project references and only restore the specified project.
  --force                               Force all dependencies to be resolved even if the last restore was successful.
                                        This is equivalent to deleting project.assets.json.
```


```
error NU5128: - Add lib or ref assemblies for the net461 target framework
error NU5128: - Add lib or ref assemblies for the netstandard1.5 target framework
error NU5128: - Add lib or ref assemblies for the netstandard2.0 target framework
```

https://docs.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu5128#scenario-2

```xml
<PropertyGroup>
    <SuppressDependenciesWhenPacking>true</SuppressDependenciesWhenPacking>
</PropertyGroup>
```

But then lose dependency on `nng.Shared` package.

```xml
  <PropertyGroup Condition="'$(TargetFramework)'=='netstandard2.0' or '$(TargetFramework)'=='netstandard2.1'">
    <DefineConstants>FEATURE_NETSTANDARD2_0_AND_UP</DefineConstants>
  </PropertyGroup>
```

https://docs.microsoft.com/en-us/nuget/quickstart/create-and-publish-a-package-using-the-dotnet-cli

To sign a package you [still need](https://github.com/NuGet/Home/issues/7939) nuget:

subor
oy2eqtpuqj6yez2sqwia377k27ifbjuupjmt63xdevjj4i
jeikabu
oy2cl32afob6kyvehi2qkgfu3pzok53eflucgzxwdwrc7i

Signing and uploading is largely unchanged [from before]({% post_url /2018/2018-08-18-nuget-sign-upload %}).

```powershell
# Create release nupkg
# -c : --configuration
dotnet pack -c Release

# Sign nupkg
nuget.exe sign .\bin\Release\Subor.nng.NETCore.1.1.1.1.nupkg -Timestamper http://sha256timestamp.ws.symantec.com/sha256/timestamp -CertificatePath path_to_cert.pfx

# Publish to nuget.org
dotnet nuget push .\bin\Release\Subor.nng.NETCore.1.1.1.1.nupkg -k <NugetApiKey> -s https://api.nuget.org/v3/index.json
```

`<NugetApiKey>` is nuget.org API key.






https://github.com/aspnet/Tooling/blob/master/missing-template.md
https://github.com/dotnet/templating/wiki/Available-templates-for-dotnet-new