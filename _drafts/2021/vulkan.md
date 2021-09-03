---
layout: post
title: Vulkan on Windows
tags:
- windows
- vulkan
- graphics
- vcpkg
- win10
- winrt
---

https://vulkan-tutorial.com/

```powershell
$env:VK_INSTANCE_LAYERS="VK_LAYER_LUNARG_standard_validation"
$env:VK_INSTANCE_LAYERS="VK_LAYER_LUNARG_api_dump;VK_LAYER_LUNARG_core_validation"
```

- No C++ exceptions.  Exceptions are rarely enabled in game engines; they're disabled by default in UE4, CryEngine/Lumberyard/O3DE, and probably 


https://vulkan.lunarg.com/doc/sdk/1.2.182.0/windows/layer_configuration.html

__Properties > VC++ Directories > Include Directories__
`$(VULKAN_SDK)\Include\vulkan`

Don't let it prepend `#pragma once` in vkfunctions.inl.


UWP win10 console

https://stackoverflow.com/questions/57219954
https://www.codeproject.com/Articles/612/Creating-a-Console-for-Your-MFC-Apps-Debug-Output

Make sure to install latest driver for your GPU to ensure has latest Vulkan support.

WinMain
https://docs.microsoft.com/en-us/windows/win32/learnwin32/winmain--the-application-entry-point

HInstance from GetModuleHandle(null)
https://stackoverflow.com/questions/1749972/determine-the-current-hinstance

winrt and directX
https://docs.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/consume-com
https://docs.microsoft.com/en-us/windows/uwp/gaming/optimize-performance-for-windows-store-direct3d-11-apps-with-coredispatcher
https://docs.microsoft.com/en-us/windows/uwp/gaming/tutorial--create-your-first-uwp-directx-game

winrt get hwnd
https://github.com/microsoft/microsoft-ui-xaml/issues/2828

VK_NV_acquire_winrt_display
https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_NV_acquire_winrt_display.html

SharpVK hello triangle
https://github.com/FacticiusVir/SharpVk/blob/UWP/SharpVk/SharpVk.HelloTriangle/Program.cs


```powershell
$env:PATH="$env:LOCALAPPDATA/Programs/Python/Python39;$env:PATH"
mkdir build && cd build
# Disable tests and target 64-bit
cmake .. -DCMAKE_INSTALL_PREFIX="$(pwd)/install" -DENABLE_CTEST=0 -A x64
cmake --build . --config Release --target install
```

Setting `PATH` might be needed if you have python2 (for when it's occasionally needed) and cmake fails with:
```
CMake Error at C:/Program Files/CMake/share/cmake-3.20/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
  Could NOT find PythonInterp: Found unsuitable version "2.7.17", but
  required is at least "3" (found C:/Python27/python.exe)
```

```
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.bat
./vcpkg integrate install
./vcpkg install glslang glslang:x64-windows
```

What we actually want is `glslang:x64-uwp`, but that fails with:
```
The BaseOutputPath/OutputPath property is not set for project 'VCTargetsPath.vcxproj'.  Please check to make sure that you have specified a valid combination of Configuration and Platform for this project.  Configuration='Debug'  Platform='x64'.  ...
```

```powershell
.\vcpkg integrate project
# In VS > Tools > NuGet Package Manager > Package Manager Console
Install-Package vcpkg.D.projects.vcpkg -Source "D:\projects\vcpkg\scripts\buildsystems"
# Set `x64-windows` in project > Properties > vcpkg > Triplet
```

```xml
<!-- In the project -->
<Import Project="..\packages\vcpkg.D.projects.vcpkg.1.0.0\build\native\vcpkg.D.projects.vcpkg.targets" ... />
<!-- In vcpkg.D.projects.vcpkg.targets -->
<Import Project="$(VCPKG_ROOT)\scripts\buildsystems\msbuild\vcpkg.targets" ... />
```
https://github.com/KhronosGroup/glslang
https://github.com/KhronosGroup/glslang/pull/2038

Designated initialization
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0329r4.pdf

`WM_SIZE`
https://docs.microsoft.com/en-us/windows/win32/winmsg/wm-size