# 01.4 — The Common Helper Files

**Read time:** ~8 minutes  
**Prev:** [Android Manifest →](03-android-manifest.md) | **Next:** [android_main Entry Point →](05-android-main.md)

---

The tutorial provides several shared helper files in `Common/`. You don't need to write these — you download them — but you should understand what each one does, because you'll use them throughout.

---

## `DebugOutput.h`

Redirects `std::cout` and `std::cerr` to Android Logcat via `__android_log_write()`.

```cpp
DebugOutput debugOutput; // construct once at the start of android_main
// Now std::cout << "something" goes to Logcat
```

Android's `printf` doesn't show up in Logcat by default. This class fixes that. Instantiate it at the very start of `android_main`.

---

## `HelperFunctions.h`

Boilerplate utilities:

- **`DEBUG_BREAK`** macro — stops execution at the point of error (uses `__builtin_trap()` on Android)
- **`IsStringInVector()`** — checks if a `const char*` exists in a `std::vector<const char*>` using `strcmp`
- **`BitwiseCheck()`** — checks if a bit is set in a bitfield

These are small but used frequently throughout the tutorial code.

---

## `OpenXRHelper.h`

The most important helper. It:

1. Includes `openxr/openxr.h` and `openxr/openxr_platform.h`
2. Defines the **`OPENXR_CHECK` macro** — your error checking wrapper
3. Defines `GetXRErrorString()` and `OpenXRDebugBreak()`

### OPENXR_CHECK

```cpp
OPENXR_CHECK(xrCreateInstance(&createInfo, &m_xrInstance), "Failed to create XrInstance.");
```

Internally this calls the OpenXR function, checks the `XrResult`, and if it's not `XR_SUCCESS`, logs the error string and calls `DEBUG_BREAK`. Use it around every OpenXR call while learning.

### Platform macros

`OpenXRHelper.h` also expects you to have defined `XR_USE_GRAPHICS_API_VULKAN` and `XR_USE_PLATFORM_ANDROID` before including it. `GraphicsAPI.h` handles this for you based on the CMake compile definition `XR_TUTORIAL_USE_VULKAN`.

---

## `OpenXRDebugUtils.h` / `OpenXRDebugUtils.cpp`

Helpers for `XR_EXT_debug_utils`:

- `CreateOpenXRDebugUtilsMessenger()` — creates the `XrDebugUtilsMessengerEXT`
- `DestroyOpenXRDebugUtilsMessenger()` — cleans it up
- `OpenXRMessageCallbackFunction()` — the callback that receives error/warning messages from the validation layer

You call Create right after `xrCreateInstance` and Destroy just before `xrDestroyInstance`. Covered in detail in [02.2 Debug Utils](../02-openxr-core/02-debug-utils.md).

---

## `GraphicsAPI.h` / `GraphicsAPI.cpp`

An abstraction layer over different graphics APIs (Vulkan, OpenGL ES, D3D). For this guide we only care about the Vulkan variant.

Key things it defines:
- `GraphicsAPI_Type` enum (`VULKAN`, `OPENGL_ES`, `D3D11`, etc.)
- `GetGraphicsAPIInstanceExtensionString()` — returns the right OpenXR extension name for your graphics API (`XR_KHR_vulkan_enable2`)
- Base `GraphicsAPI` class interface

---

## `GraphicsAPI_Vulkan.h` / `GraphicsAPI_Vulkan.cpp`

The Vulkan implementation of `GraphicsAPI`. Handles:
- Creating the `VkInstance` and `VkDevice` in the way OpenXR requires
- Managing Vulkan resources (pipelines, render passes, framebuffers)
- The `RenderCuboid()` helper used in Chapter 3

This is a significant chunk of code but it's provided so you can focus on the OpenXR side, not on vanilla Vulkan boilerplate.

---

## Include Order

In your `main.cpp`:

```cpp
#include <DebugOutput.h>         // first — redirects stdout
#include <GraphicsAPI_Vulkan.h>  // pulls in GraphicsAPI.h, OpenXRHelper.h, HelperFunctions.h
#include <OpenXRDebugUtils.h>    // debug messenger helpers
```

`GraphicsAPI_Vulkan.h` transitively includes everything else you need.

---

**Next:** [android_main and the App Entry Point →](05-android-main.md)