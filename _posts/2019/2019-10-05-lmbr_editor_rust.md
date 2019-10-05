---
layout: post
title: Lumberyard Editor Plugin in Rust
tags:
- lumberyard
- gamedev
- rust
canonical_url: 
series: Rust in Lumberyard
---

With a [Rust static library working in Lumberyard]({% post_url /2019/2019-09-30-lmbr_rust %}), let's try something more interesting.  The main content creation tool, [__Editor__](https://docs.aws.amazon.com/lumberyard/latest/gettingstartedguide/intro.html), has a plugin framework that would be an ideal way to gradually introduce Rust as part of normal development.

## Editor Plugins

We haven't yet found documentation in the Lumberyard User Guide on Editor plugins, but it's pretty easy to figure out what's going on.

__TL;DR__:  
If we place a shared library in `EditorPlugins/` that exports `IPlugin* CreatePluginInstance(PLUGIN_INIT_PARAM*)`, we can get Editor to load and register our plugin.

Open one of the `wscript` files in `Code/Sandbox/Plugins/` and you'll see they use either the `CryPlugin` or `CryStandAlonePlugin` [Waf module type](https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-using-module.html) (defined in `dev\Tools\build\waf-1.7.13\lmbrwaflib\cryengine_modules.py`).  Plugin binaries are output to `dev/Bin64vc141/EditorPlugins/`, and if we take a look at the exported symbols:
```powershell
dumpbin /exports Bin64vc141/EditorPlugins/MetricsPlugin.dll
# Output
    1    0 000023E0 CreatePluginInstance = CreatePluginInstance
    2    1 00002430 DetachEnvironment = DetachEnvironment
    3    2 00002460 InjectEnvironment = InjectEnvironment
    4    3 00002480 ModuleInitISystem = ModuleInitISystem
    5    4 00002430 ModuleShutdownISystem = DetachEnvironment
```

For what happens at runtime, look at [`Sandbox/Editor/PluginManager.cpp`](https://github.com/aws/lumberyard/blob/e881f3023cc1840650eb7b133e605881d1d4330d/dev/Code/Sandbox/Editor/PluginManager.cpp#L184) (abridged here):
```cpp
// 1. Load the shared library
QLibrary *hPlugin = new QLibrary(iter->m_path);
AZStd::string pathUtf8(iter->m_path.toUtf8().constData());
if (!hPlugin->load())
{
    continue;
}

// 2. Get the address of `CreatePluginInstance()`
TPfnCreatePluginInstance pfnFactory = reinterpret_cast<TPfnCreatePluginInstance>(hPlugin->resolve("CreatePluginInstance"));

if (!pfnFactory)
{
    continue;
}

IPlugin* pPlugin = NULL;
PLUGIN_INIT_PARAM sInitParam = /* ... */;

try
{
    // 3. Call `CreatePluginInstance()` to obtain `IPlugin*` to plugin object
    pPlugin = SafeCallFactory(pfnFactory, &sInitParam, iter->m_path.toUtf8().data());
}
catch (...)
{
    continue;
}
```

## Rust vs Polymorphism

__TL;DR__:  
There's no built-in support for passing objects much more complex than POD through Rust/C++ FFI.

Let's take a look at `IPlugin` and other defines in `Sandbox/Editor/Include/IPlugin.h`
```cpp
struct IPlugin
{
    enum EError
    {
        eError_None = 0,
        eError_VersionMismatch = 1
    };

    virtual ~IPlugin() = default;
    
    // Releases plugin.
    virtual void Release() = 0;
    //! Show a modal about dialog / message box for the plugin
    virtual void ShowAbout() = 0;
    //! Return the GUID of the plugin
    virtual const char* GetPluginGUID() = 0;
    virtual DWORD GetPluginVersion() = 0;
    //! Return the human readable name of the plugin
    virtual const char* GetPluginName() = 0;
    // <SNIP>
};

// Initialization structure
struct PLUGIN_INIT_PARAM
{
    IEditor* pIEditorInterface;
    int pluginVersion;
    IPlugin::EError outErrorCode;
};

// <SNIP>

// Factory API
extern "C"
{
PLUGIN_API IPlugin* CreatePluginInstance(PLUGIN_INIT_PARAM* pInitParam);
PLUGIN_API void QueryPluginSettings(SPluginSettings& settings);
}
```

`virtual` functions.  If you've worked in software a few years you've invariably run across an interview question about implementing [vtables](https://en.wikipedia.org/wiki/Virtual_method_table).  The story in Rust:

- Rust has [Trait Objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html) for dynamic dispatch
- [Bindgen doesn't support them](https://github.com/rust-lang/rust-bindgen/issues/27) (yet)
- Rust FFI prefers forwarding to C methods (see [FFI Omnibus](http://jakegoulding.com/rust-ffi-omnibus/objects/), [use in Firefox](https://hsivonen.fi/modern-cpp-in-rust/), etc.)
- There's crates like [vptr](https://docs.rs/vptr/0.2.1/vptr/) with alternative implementations

## Implementation

We ever-so-briefly contemplated manually building the vtable so `IPlugin` instances could be completely constructed in Rust.  Instead, we'll go the less brittle route and do C++ object creation in C++ via a "EditorRustPlugin" static library:

```cpp
#include <platform_impl.h>
#include "EditorRustPlugin.h"
#include <Include/IPlugin.h>

extern "C" PLUGIN_API IPlugin* CreatePluginInstance(PLUGIN_INIT_PARAM* pInitParam)
{
    if (pInitParam->pluginVersion != SANDBOX_PLUGIN_SYSTEM_VERSION)
    {
        pInitParam->outErrorCode = IPlugin::eError_VersionMismatch;
        return 0;
    }

    //ISystem* pSystem = pInitParam->pIEditorInterface->GetSystem();
    //ModuleInitISystem(pSystem, rust_editor_plugin_module_name());

    return new CEditorRustPlugin(pInitParam->pIEditorInterface);
}
```

Where `CEditorRustPlugin` implements `IPlugin` and calls C functions exported by our Rust:
```cpp
#include <Include/IPlugin.h>

// Defined in our Rust library
extern "C" {
    void rust_editor_plugin_init(void* p_iSystem);
    const char* rust_editor_plugin_module_name();
    const char* rust_editor_plugin_guid();
    int rust_editor_plugin_version();
    const char* rust_editor_plugin_name();
    bool rust_editor_plugin_can_exit_now();
    void rust_editor_plugin_release();
}

// IPlugin implementation that calls C API
class CEditorRustPlugin : public IPlugin
{
public:
	CEditorRustPlugin(IEditor* pIEditorInterface);

	void Release() override
	{
		rust_editor_plugin_release();
	}
	void ShowAbout() override
	{

	}
	const char* GetPluginGUID() override
	{
		return rust_editor_plugin_guid();
	}
	DWORD GetPluginVersion() override
	{
		return rust_editor_plugin_version();
	}
	const char* GetPluginName() override
	{
		return rust_editor_plugin_name();
	}
	bool CanExitNow() override
	{
		return rust_editor_plugin_can_exit_now();
	}
	void OnEditorNotify(EEditorNotifyEvent aEventId) override
	{

	}
};
```

Now the Rust side of the plugin.  First, our `Cargo.toml`:
```toml
[package]
name = "editor_plugin"
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"]

[features]
default = ["lmbr_sys"]

[build-dependencies]
lmbr_build = {version = "0.1", path = "../lmbr_build"}

[dependencies]
lmbr_logger = { version = "0.1", path = "../lmbr_logger" }
lmbr_sys = { version = "0.1", optional = true, path = "../lmbr_sys" }
log = "0.4"
```

Arguably the most important part is setting [crate type](https://doc.rust-lang.org/reference/linkage.html) to `cdylib` so we're producing a shared library (*.dll).

In `lib.rs`:
```rust
use log::info;
use std::os::raw::{c_char, c_void};

const MODULE_NAME: &[u8] = b"Rust Editor Plugin Example\0";
const GUID: &[u8] = b"{98C1DC36-5D1E-4CF6-91CE-AFA1117CE81F}\0";

#[no_mangle]
pub extern fn rust_editor_plugin_init(p_isystem: *mut c_void) {
    lmbr_logger::init().unwrap();
    unsafe {
        lmbr_sys::ModuleInitISystem(p_isystem, rust_editor_plugin_module_name());
    }
    info!("Initialized");
}

#[no_mangle]
pub extern fn rust_editor_plugin_module_name() -> *const c_char {
    MODULE_NAME.as_ptr() as *const _
}

#[no_mangle]
pub extern fn rust_editor_plugin_guid() -> *const c_char {
    GUID.as_ptr() as *const _
}

#[no_mangle]
pub extern fn rust_editor_plugin_version() -> u32 {
    1
}

#[no_mangle]
pub extern fn rust_editor_plugin_name() -> *const c_char {
    rust_editor_plugin_module_name() as *const _
}

#[no_mangle]
pub extern fn rust_editor_plugin_can_exit_now() -> bool {
    true
}

#[no_mangle]
pub extern fn rust_editor_plugin_release() {
}
```

Note how we're creating C-compatible null-terminated string literals with `b"XXX\0"`.

`ModuleInitISystem()` needs to be called by each plugin to initialize globals and so on.  In practice, we should call this from `CreatePluginInstance()` in the EditorRustPlugin C++ library since it's shared by all Rust plugins, but we've got it here just to test calling some other C/C++ methods.

In `build.rs`:
```rust
use lmbr_build::*;
use std::{path::PathBuf};

fn main() {
    // Helpers in lmbr_build to:
    // - Map Lumberyard build configs to Rust build configs
    // - Locate Lumberyard install path
    // - Construct paths to Lumberyard resources
    let config = build_config();
    let config_3rdparty = if config == BuildConfig::Debug { "debug" } else { "release" };
    let ly_root = PathBuf::from(lmbr_root().unwrap());
    let bintemp = ly_root.join("dev/BinTemp").join(bintemp_dir());
    let bintemp_fw = bintemp.join("Code/Framework");
    println!(
        "cargo:rustc-link-search=native={}",
        bintemp_fw.join("AzCore/AzCore").display()
    );
    println!(
        "cargo:rustc-link-search=native={}",
        bintemp_fw.join("AzFramework/AzFramework").display()
    );
    println!("cargo:rustc-link-search=native={}",
        ly_root.join(r"3rdParty\Lua\5.1.1.8-az\build\win_x64\vc140").join(config_3rdparty).display()
    );
    println!("cargo:rustc-link-search=native={}",
        ly_root.join(r"3rdParty\zlib\1.2.8-pkg.2\build\win_x64\vc140").join(config_3rdparty).display()
    );
    println!("cargo:rustc-link-search=native={}",
        ly_root.join("dev").join(bin_dir()).display()
    );
    println!("cargo:rustc-link-search=native={}",
        bintemp.join("Code/Sandbox/EditorRustPlugin").display()
    );
    
    println!("cargo:rustc-link-lib=static=AzCore");
    println!("cargo:rustc-link-lib=static=AzFramework");
    println!("cargo:rustc-link-lib=static=lua");
    println!("cargo:rustc-link-lib=static=zlib");
    println!("cargo:rustc-link-lib=static=user32");
    println!("cargo:rustc-link-lib=static=EditorRustPlugin");
}
```

This is mostly linking with Lumberyard dependencies as well as with "EditorRustPlugin" from above.

`cargo build` and if we check the functions exported by the resulting dll it certainly looks like it will satisfy Editor:
```powershell
dumpbin /exports .\Bin64vc141\EditorPlugins\editor_plugin.dll
# Output
    1    0 0001C930 CreatePluginInstance
    2    1 0001C970 DetachEnvironment
    3    2 0001C9A0 InjectEnvironment
    4    3 0001C9C0 ModuleInitISystem
    5    4 0001CB20 ModuleShutdownISystem
    6    5 00001230 rust_editor_plugin_can_exit_now
    7    6 000011D0 rust_editor_plugin_guid
    8    7 00001060 rust_editor_plugin_init
    9    8 000011A0 rust_editor_plugin_module_name
    10    9 00001210 rust_editor_plugin_name
    11    A 00001240 rust_editor_plugin_release
    12    B 00001200 rust_editor_plugin_version
    13    C 0000EA40 rust_eh_personality
```

Launch Editor and sure enough:  
![](/assets/lmbr_editor_plugin_console.png)

Very cool.  Rust in all the things!

Something I get a kick out of, with Rust being "just native code" Visual Studio can step into and through it without plugins/extensions:  
![](/assets/lmbr_vs_debug_rust.gif)

## Common Problems

If you get `error LNK2001: unresolved external symbol` for `CryAssert*()` methods:
```
EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
```

Make sure at least one of the Lumberyard source files contains:
```c
#include <platform_impl.h>
```

In Editor, if `LoadLibrary()` returns null and `GetLastError()` is `193` (`0xC1`- `ERROR_BAD_EXE_FORMAT`) make sure you're building/running Lumberyard for `x64`, not `x86`.  32-bit applications can't load 64-bit shared libraries (nor vice-versa).

## Next Steps

We're a little surprised linking to "EditorRustPlugin" works.  According to [this open issue](https://github.com/rust-lang/rust/issues/36342), re-exporting C symbols in a `cdylib` doesn't work.  This might break unexpectedly and we'd like to better understand what's going on here.

There's yet [another issue](https://github.com/rust-lang/rust-bindgen/issues/1609) with bindgen failing on Lumberyard's more convoluted templates.  Ultimately, bindgen's [short-comings](https://rust-lang.github.io/rust-bindgen/cpp.html#unsupported-features) and pain related to polymorphic types may limit what we can do in Rust without exerting more effort.
