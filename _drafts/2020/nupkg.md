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


Signing and uploading is largely unchanged [from before]({% post_url /2018/2018-08-18-nuget-sign-upload %}).




RUYI@20!8
