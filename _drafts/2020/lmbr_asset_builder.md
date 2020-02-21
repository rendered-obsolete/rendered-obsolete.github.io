---
layout: post
title: Lumberyard Asset Builder in Rust
tags:
- lumberyard
- gamedev
- rust
- ffi
- bindgen
series: Rust in Lumberyard
---

Lumberyard's __AssetBuilderSDK__ can be used to process a custom asset type for use at runtime.  ["Creating a Custom Asset Builder" in the User Guide](https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-builder-custom.html) shows how to do this in C++.  This is another extension point using shared libraries which could potentially be written in Rust.

For a concrete example, look at __MaterialBuilder__ found in `Code\Tools\AssetProcessor\Builders\MaterialBuilder\` because it's one of the simplest.  [At the bottom of `MaterialBuilder\Source\MaterialBuilderApplication.cpp`](https://github.com/aws/lumberyard/blob/v1.22.0.0/dev/Code/Tools/AssetProcessor/Builders/MaterialBuilder/Source/MaterialBuilderApplication.cpp#L36) you'll find the important sounding `REGISTER_ASSETBUILDER` macro.

It's defined in [`Code\Tools\AssetProcessor\AssetBuilderSDK\AssetBuilderSDK\AssetBuilderSDK.h`](https://github.com/aws/lumberyard/blob/07228c605ce16cbf5aaa209a94a3cb9d6c1a4115/dev/Code/Tools/AssetProcessor/AssetBuilderSDK/AssetBuilderSDK/AssetBuilderSDK.h):
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

Asset builders need to export particular functions.  We can use the same approach as with [Editor plugins]({% post_url /2019/2019-10-05-lmbr_editor_rust %}) to register a builder written in Rust:

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

The extra `bool` is from the default argument in [`Code\Framework\AzCore\AzCore\Module\Environment.h`](https://github.com/aws/lumberyard/blob/v1.22.0.0/dev/Code/Framework/AzCore/AzCore/Module/Environment.h#L143):
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

`Cargo.toml`:
```toml
[package]
name = "asset_builder_plugin"
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"] # Produce shared library (.dll)

[build-dependencies]
lmbr_build = {version = "0.1", path = "../lmbr_build"}

[dependencies]
lmbr_logger = { version = "0.1", path = "../lmbr_logger" }
lmbr_sys = { version = "0.1", path = "../lmbr_sys" }
log = "0.4"
```

`cargo build` and check the binary with `dumpbin /exports <path>\target\debug\asset_builder_plugin.dll` (in Visual Studio shell):
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

Copy `target/debug/asset_builder_plugin.dll` to Lumberyard's `Bin64vc141/Builders/` folder.  Launch __AssetProcessor__ (`Bin64vc141/AssetProcessor.exe`), and in __Logs > Messages__:
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

The first is [working with ebus like we investigated previously]({% post_url /2019/2019-12-06-lmbr_ebus %}) and easy enough to do from Rust.  First, wrap the C++:
```cpp
void AssetBuilderBus_Broadcast_RegisterComponentDescriptor(AZ::ComponentDescriptor* descriptor)
{
    AssetBuilderSDK::AssetBuilderBus::Broadcast(&AssetBuilderSDK::AssetBuilderBus::Events::RegisterComponentDescriptor, descriptor);
}
```
Then, call the wrapped method from Rust:
```rust
// In lmbr_sys crate
extern "C" {
    pub fn AssetBuilderBus_Broadcast_RegisterComponentDescriptor(descriptor: *mut AZ::ComponentDescriptor);
}

// Elsewhere
let descriptor: *mut AZ::ComponentDescriptor = /* ... */;
unsafe {
    lmbr_sys::AssetBuilderBus_Broadcast_RegisterComponentDescriptor(descriptor);
}
```

MaterialBuilder calls `CreateComponentIfReady()`, but if we look at the [definition in `Entity.h`](https://github.com/aws/lumberyard/blob/v1.22.0.0/dev/Code/Framework/AzCore/AzCore/Component/Entity.h#L202):
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

It's not immediately clear to me how to replace it with `AzToolsFramework::EntityCompositionRequestBus`, but it should be ok for our purposes.

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

## Bad Strings

`AZStd::string` is defined in [`string.h`](https://github.com/aws/lumberyard/blob/07228c605ce16cbf5aaa209a94a3cb9d6c1a4115/dev/Code/Framework/AzCore/AzCore/std/string/string.h).  An abridged version:

```cpp
typedef basic_string<char >     string;

// In allocator.h
class allocator
{
public:

    AZ_TYPE_INFO(allocator, "{E9F5A3BE-2B3D-4C62-9E6B-4E00A13AB452}");

    typedef void*               pointer_type;
    typedef AZStd::size_t       size_type;
    typedef AZStd::ptrdiff_t    difference_type;

    //...
}

// In string.h
template<class Element, class Traits = char_traits<Element>, class Allocator = AZStd::allocator >
class basic_string
{
public:
    typedef Element*                                pointer;
    typedef const Element*                          const_pointer;

    typedef Element&                                reference;
    typedef const Element&                          const_reference;
    typedef typename Allocator::difference_type     difference_type;
    typedef typename Allocator::size_type           size_type;

    //...

protected:
    enum
    {   // length of internal buffer, [1, 16]
        SSO_BUF_SIZE = 16 / sizeof (Element) < 1 ? 1 : 16 / sizeof(Element)
    };
    enum
    {   // roundup mask for allocated buffers, [0, 15]
        _ALLOC_MASK = sizeof (Element) <= 1 ? 15 : sizeof (Element) <= 2 ? 7 : sizeof (Element) <= 4 ? 3 : sizeof (Element) <= 8 ? 1 : 0
    };

    //...

    union //Storage
    {
        Element m_buffer[SSO_BUF_SIZE];     //< small buffer used for small string optimization
        pointer m_data;                     //< dynamically allocated data
    };

    size_type m_size;           // current length of string
    size_type m_capacity;       // current storage reserved for string
    allocator_type m_allocator;
}
```

- For `char`, `SSO_BUF_SIZE` is 16 so `m_buffer` provides 16 bytes of inline storage for short strings.  Longer strings use `m_data` and a heap allocation.
- `size_type` and `difference_type` are `size_t` and `ptrdiff_t`, respectively.
- When targetting 64-bit and ignoring alignment/padding, `sizeof(string) == 40` (bytes).

Bindgen outputs:
```rs
pub struct basic_string<Element, Allocator> {
    pub __bindgen_anon_1: root::AZStd::basic_string__bindgen_ty_3<Element>,
    pub m_size: root::AZStd::basic_string_size_type,
    pub m_capacity: root::AZStd::basic_string_size_type,
    pub m_allocator: root::AZStd::basic_string_allocator_type<Allocator>,
    pub _phantom_0: ::std::marker::PhantomData<::std::cell::UnsafeCell<Element>>,
    pub _phantom_1: ::std::marker::PhantomData<::std::cell::UnsafeCell<Allocator>>,
}
//...
pub type basic_string_difference_type = [u8; 0usize];
pub type basic_string_size_type = [u8; 0usize];
//...
pub const basic_string_SSO_BUF_SIZE: root::AZStd::basic_string__bindgen_ty_1 =
    basic_string__bindgen_ty_1::SSO_BUF_SIZE;
#[repr(i32)]
#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash)]
pub enum basic_string__bindgen_ty_1 {
    SSO_BUF_SIZE = 0,
}
pub const basic_string__ALLOC_MASK: root::AZStd::basic_string__bindgen_ty_2 =
    basic_string__bindgen_ty_2::_ALLOC_MASK;
#[repr(i32)]
#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash)]
pub enum basic_string__bindgen_ty_2 {
    _ALLOC_MASK = 0,
}
#[repr(C)]
pub union basic_string__bindgen_ty_3<Element> {
    pub m_buffer: *mut Element,
    pub m_data: root::AZStd::basic_string_pointer<Element>,
    _bindgen_union_align: u64,
    pub _phantom_0: ::std::marker::PhantomData<::std::cell::UnsafeCell<Element>>,
}
```

- `SSO_BUF_SIZE` is `0` and `m_buffer` inline storage is just a pointer `*mut Element`
- `basic_string::size_type` and `basic_string::different_type` are `[u8; 0usize]`- array of __0__ bytes
- Ignoring alignment/padding, `sizeof(basic_string) == 16` (`std::mem::size_of::<[u8; 0]>() == 0` and `PhantomData` is also [zero-sized](https://doc.rust-lang.org/stable/std/marker/struct.PhantomData.html))

An `AZStd::string` returned from C++ was ok, but as soon as you did anything with it in Rust you'd get invalid memory access exceptions.  This wasn't Rust's fault, we lied about the size and contents of the type.

We can manually fix it by changing the following lines:
```rs
pub type basic_string_difference_type = u64;
pub type basic_string_size_type = u64;
//...
pub const basic_string_SSO_BUF_SIZE: root::AZStd::basic_string__bindgen_ty_1 =
    basic_string__bindgen_ty_1::SSO_BUF_SIZE;
#[repr(i32)]
#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash)]
pub enum basic_string__bindgen_ty_1 {
    SSO_BUF_SIZE = 16,
}

#[repr(C)]
pub union basic_string__bindgen_ty_3<Element> {
    pub m_buffer: [u8; basic_string_SSO_BUF_SIZE as usize],
    //...
}
```

Now `string` is usable from Rust.

## Closing

The remainder of fleshing out our asset builder scaffolding involves calling bindgen generated bindings from `lmbr_sys` and creating wrappers for stickier C++ functions.

Wrappers are getting common enough I need to split them out from `lmbr_sys` so it only contains bindings and another crate contains the various "helpers" and utility functions.  Perhaps another day.