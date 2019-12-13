---
layout: post
title: Lumberyard Asset Builder from Rust
tags:
- lumberyard
- gamedev
- rust
series: Rust in Lumberyard
---

https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-builder-custom.html

__MaterialBuilder__ found in `Code\Tools\AssetProcessor\Builders\MaterialBuilder\` because it's one of the simplest.

At the bottom of `MaterialBuilder\Source\MaterialBuilderApplication.cpp` you'll find the important sounding `REGISTER_ASSETBUILDER` macro.

It's defined in `Code\Tools\AssetProcessor\AssetBuilderSDK\AssetBuilderSDK\AssetBuilderSDK.h`:
```cpp
//! This macro should be used by every AssetBuilder to register itself,
//! AssetProcessor uses these exported function to identify whether a dll is an Asset Builder or not
//! If you want something highly custom you can do these entry points yourself instead of using the macro.
#define REGISTER_ASSETBUILDER                                                      \
    extern void BuilderOnInit();                                                   \
    extern void BuilderDestroy();                                                  \
    extern void BuilderRegisterDescriptors();                                      \
    extern void BuilderAddComponents(AZ::Entity * entity);                         \
    extern "C"                                                                     \
    {                                                                              \
    AZ_DLL_EXPORT int IsAssetBuilder()                                             \
    {                                                                              \
        return 0;                                                                  \
    }                                                                              \
                                                                                   \
    AZ_DLL_EXPORT void InitializeModule(AZ::EnvironmentInstance sharedEnvironment) \
    {                                                                              \
        AZ::Environment::Attach(sharedEnvironment);                                \
        BuilderOnInit();                                                           \
    }                                                                              \
                                                                                   \
    AZ_DLL_EXPORT void UninitializeModule()                                        \
    {                                                                              \
        BuilderDestroy();                                                          \
        AZ::Environment::Detach();                                                 \
    }                                                                              \
                                                                                   \
    AZ_DLL_EXPORT void ModuleRegisterDescriptors()                                 \
    {                                                                              \
        BuilderRegisterDescriptors();                                              \
    }                                                                              \
                                                                                   \
    AZ_DLL_EXPORT void ModuleAddComponents(AZ::Entity * entity)                    \
    {                                                                              \
        BuilderAddComponents(entity);                                              \
    }                                                                              \
    }
// confusion-reducing note: above end-brace is part of the macro, not a namespace
```

We saw how to accomplish this when creating an [Editor plugin]({% post_url /2019/2019-10-05-lmbr_editor_rust %}).

```rust
// Ignore warnings like:
// warning: function `IsAssetBuilder` should have a snake case name
#![allow(non_snake_case)]

use lmbr_sys::root::AZ;

#[no_mangle]
pub fn IsAssetBuilder() -> i32 {
    0
}

#[no_mangle]
pub fn InitializeModule(shared_environment: AZ::EnvironmentInstance) {
    unsafe {
        AZ::Environment::Attach(shared_environment, false);
    }
    // Implement custom init
}

#[no_mangle]
pub fn UninitializeModule() {
    // Implement custom uninit

    unsafe {
        AZ::Environment::Detach();
    }
}

#[no_mangle]
pub fn ModuleRegisterDescriptors() {
    // Implement
}

#[no_mangle]
pub fn ModuleAddComponents(entity: *mut AZ::Entity) {
    // Implement
}
```

Notice how in Rust `Environment::Attach()` takes two arguments.  Bindgen generates the following for `AZ::Environment::Attach()`:
```rust
pub mod Environment {
    //...
    extern "C" {
        #[link_name = "\u{1}?Attach@Environment@AZ@@YAXPEAVEnvironmentInterface@Internal@2@_N@Z"]
        pub fn Attach(
            sourceEnvironment: root::AZ::EnvironmentInstance,
            useAsGetFallback: bool,
        );
    }
}
```

The extra `bool` is from the default argument in `Code\Framework\AzCore\AzCore\Module\Environment.h`:
```cpp
/**
    * Attaches the current module environment from sourceEnvironment.
    * note: this is not a copy it will actually reference the source environment, so any variables
    * you add remove will be visible to all shared modules
    * \param useAsFallback if set to true a new environment will be created and only failures to GetVariable
    * which check the shared environment. This way you can change the environment.
    */
void Attach(EnvironmentInstance sourceEnvironment, bool useAsGetFallback = false);
```

```toml
[package]
name = "asset_builder_plugin"
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"]

[build-dependencies]
lmbr_build = {version = "0.1", path = "../lmbr_build"}

[dependencies]
lmbr_logger = { version = "0.1", path = "../lmbr_logger" }
lmbr_sys = { version = "0.1", path = "../lmbr_sys" }
log = "0.4"
```

`dumpbin /exports <path>\target\debug\asset_builder_plugin.dll`:
```
Microsoft (R) COFF/PE Dumper Version 14.23.28106.4
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file D:\projects\lmbr\target\debug\asset_builder_plugin.dll

File Type: DLL

  Section contains the following exports for asset_builder_plugin.dll

    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
           6 number of functions
           6 number of names

    ordinal hint RVA      name

          1    0 00001010 InitializeModule = InitializeModule
          2    1 00001000 IsAssetBuilder = IsAssetBuilder
          3    2 00001050 ModuleAddComponents = ModuleAddComponents
          4    3 00001040 ModuleRegisterDescriptors = ModuleRegisterDescriptors
          5    4 00001030 UninitializeModule = UninitializeModule
          6    5 00001200 rust_eh_personality = rust_eh_personality

  Summary

       5A000 .data
        A000 .pdata
       33000 .rdata
        2000 .reloc
       6C000 .text
        3000 _RDATA
```

Copy `target/debug/asset_builder_plugin.dll` to Lumberyard's `Bin64vc141/Builders/` folder.  Launch __AssetProcessor.exe__, and in __Logs > Messages__:
```
12/12/2019 4:09 PM - AssetProcessor - Initializing and registering builder D:/projects/lumberyard/dev/Bin64vc141/Builders/asset_builder_plugin.dll
```

Looks good so far.

## Component

The implementation for `ModuleRegisterDescriptors()` and `ModuleAddComponents()` are deferred to `BuilderRegisterDescriptors()` and `BuilderAddComponents()`, respectively:
```cpp
void BuilderRegisterDescriptors()
{
    EBUS_EVENT(AssetBuilderSDK::AssetBuilderBus, RegisterComponentDescriptor, MaterialBuilder::BuilderPluginComponent::CreateDescriptor());
}

void BuilderAddComponents(AZ::Entity* entity)
{
    entity->CreateComponentIfReady<MaterialBuilder::BuilderPluginComponent>();
}
```

The first is easy enough [we already investigated using Ebus]({% post_url /2019/2019-12-06-lmbr_ebus %}):
```cpp
void AssetBuilderBus_Broadcast_RegisterComponentDescriptor(AZ::ComponentDescriptor* descriptor)
{
    AssetBuilderSDK::AssetBuilderBus::Broadcast(&AssetBuilderSDK::AssetBuilderBus::Events::RegisterComponentDescriptor, descriptor);
}
```
```rust
// lmbr_sys crate
extern "C" {
    pub fn AssetBuilderBus_Broadcast_RegisterComponentDescriptor(descriptor: *mut AZ::ComponentDescriptor);
}

// Elsewhere
let descriptor: *mut AZ::ComponentDescriptor = /* ... */;
unsafe {
    lmbr_sys::AssetBuilderBus_Broadcast_RegisterComponentDescriptor(descriptor);
}
```

```cpp
/// @cond EXCLUDE_DOCS 
/**
 * @deprecated In tools, use AzToolsFramework::EntityCompositionRequestBus 
 * to ensure component requirements are met. 
 */
template<class ComponentType>
ComponentType* CreateComponentIfReady()
{
    return static_cast<ComponentType*>(CreateComponentIfReady(AzTypeInfo<ComponentType>::Uuid()));
}
/// @endcond
```

It's not immediately clear to me how this logic with `AzToolsFramework::EntityCompositionRequestBus`

## Debugging

Note that if you want to debug the AssetProcessor
- GemRegistry; only build with certain specs
- AssetProcessor/Builders; only built with "all" spec

## Logging

[Message logging](https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-builder-custom.html#asset-builder-custom-optional-implement-message-logging)

```rust
lmbr_logger::init().unwrap();
log::warn!("Hello AssetBuilder");
```

Output ends up in `Bin64vc141\logs\JobLogs\Editor\Icons\Components\Flipbook.png-525422DE-05B3-4095-966F-90CD7657A7E1_createJobs.log`:
```
~~1576157159161~~1~~0000000000001D80~~AssetBuilder~~: Hello AssetBuilder
```

```rust
unsafe {
    let uuid = "{166A7962-A3E4-4451-AC1A-AAD32E29C52C}";
    let uuid = AZ::Uuid_CreateString(uuid.as_ptr() as *const _, uuid.len());
    AssetBuilderSDK::BuilderLog(uuid, std::ffi::CString::new("Hello AssetBuilder").unwrap().as_ptr())
}
```

`Code\Tools\AssetProcessor\Builders\LuaBuilder\Source\LuaBuilderApplication.cpp`:
```cpp
AssetBuilderSDK::AssetBuilderDesc builderDescriptor;
//...
builderDescriptor.m_busId = azrtti_typeid<LuaBuilderWorker>(); // 166A7962-A3E4-4451-AC1A-AAD32E29C52C
//...
AssetBuilderSDK::AssetBuilderBus::Broadcast(&AssetBuilderSDK::AssetBuilderBusTraits::RegisterBuilderInformation, builderDescriptor);
```

`Code\Tools\AssetProcessor\Builders\LuaBuilder\Source\LuaBuilderWorker.h`
```cpp
class LuaBuilderWorker
        : public AssetBuilderSDK::AssetBuilderCommandBus::Handler
    {
    public:
        AZ_TYPE_INFO(LuaBuilderWorker, "{166A7962-A3E4-4451-AC1A-AAD32E29C52C}");
    //...
```
