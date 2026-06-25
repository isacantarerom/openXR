# 02.1 — Creating an XrInstance

> **~10 min read** | Prerequisites: 01.4 (Common Helpers), 01.5 (android_main)

An `XrInstance` is the root object of every OpenXR application. Before you can talk to a headset, query a system, or open a session, you need one. This guide walks through exactly what goes into creating it and why each piece matters.

---

## What an XrInstance Actually Is

Think of `XrInstance` as your application's contract with the OpenXR runtime. When you create it you're saying:

- "Here's who I am" (application info)
- "Here are the API layers I want" (validation, diagnostics)
- "Here are the extensions I need" (Vulkan graphics binding, debug utils, etc.)

The runtime validates all of that, allocates internal state, and hands you back an opaque handle. Everything else in OpenXR flows from that handle.

---

## XrApplicationInfo

The first thing to fill out is `XrApplicationInfo`:

```cpp
XrApplicationInfo appInfo{};
strncpy(appInfo.applicationName, "MyXRApp", XR_MAX_APPLICATION_NAME_SIZE);
appInfo.applicationVersion = 1;
strncpy(appInfo.engineName,      "None",    XR_MAX_ENGINE_NAME_SIZE);
appInfo.engineVersion        = 0;
appInfo.apiVersion           = XR_CURRENT_API_VERSION;
```

A few things worth noting:

- **`applicationName`** is a fixed-size char array — use `strncpy`, not `=`.
- **`apiVersion`** should always be `XR_CURRENT_API_VERSION` (a macro from the SDK headers). Don't hardcode a number here.
- **`engineName`** is optional context for runtime diagnostics. If you're using an engine like Unreal, put its name here. For raw apps, `"None"` or your own label is fine.

---

## Choosing API Layers

API layers sit between your app and the runtime, intercepting calls — similar to Vulkan layers. You request them by name at instance creation time.

The one you almost always want during development:

```cpp
std::vector<const char*> layers = {
    "XR_APILAYER_LUNARG_core_validation"
};
```

This layer validates your usage of the API and emits errors when you pass bad structs, call functions out of order, or use handles incorrectly. It's invaluable during development; strip it for release builds.

> **To check what layers are available** on the current runtime, call `xrEnumerateApiLayerProperties` before creating the instance. We wrap this in a helper in the Khronos tutorial's `Common/` folder.

---

## Choosing Instance Extensions

Extensions are capabilities you opt into. You must declare them at instance creation time — you can't add them later.

At minimum for an Android + Vulkan app:

```cpp
std::vector<const char*> extensions = {
    XR_KHR_ANDROID_CREATE_INSTANCE_EXTENSION_NAME,  // "XR_KHR_android_create_instance"
    XR_KHR_VULKAN_ENABLE2_EXTENSION_NAME,           // "XR_KHR_vulkan_enable2"
    XR_EXT_DEBUG_UTILS_EXTENSION_NAME,              // "XR_EXT_debug_utils"  (dev builds)
};
```

Always check that these extensions are actually supported before requesting them — use `xrEnumerateInstanceExtensionProperties`. The Khronos tutorial's `OpenXRHelper.h` has a convenience function for this.

---

## XrInstanceCreateInfo

Now assemble the create info struct:

```cpp
XrInstanceCreateInfo createInfo{XR_TYPE_INSTANCE_CREATE_INFO};
createInfo.applicationInfo         = appInfo;
createInfo.enabledApiLayerCount    = (uint32_t)layers.size();
createInfo.enabledApiLayerNames    = layers.data();
createInfo.enabledExtensionCount   = (uint32_t)extensions.size();
createInfo.enabledExtensionNames   = extensions.data();
```

Notice `{XR_TYPE_INSTANCE_CREATE_INFO}` — every OpenXR struct has a `type` field that must be set to the matching `XR_TYPE_*` enum value. This lets the runtime safely inspect and version structs. Forgetting it is a common source of confusing crashes.

---

## The Android Extension Chaining

On Android you need one extra piece: `XrInstanceCreateInfoAndroidKHR`. This is an extension struct that gets **chained** onto `createInfo` via the `next` pointer:

```cpp
XrInstanceCreateInfoAndroidKHR androidInfo{XR_TYPE_INSTANCE_CREATE_INFO_ANDROID_KHR};
androidInfo.applicationVM       = app->activity->vm;       // JavaVM*
androidInfo.applicationActivity = app->activity->clazz;    // jobject

createInfo.next = &androidInfo;
```

The `next` pointer chain is how OpenXR handles optional, extensible struct data without breaking binary compatibility. You'll see this pattern everywhere. The rule: the runtime walks the chain by type until it finds what it knows about, ignoring the rest.

---

## Calling xrCreateInstance

```cpp
XrInstance instance = XR_NULL_HANDLE;
XrResult result = xrCreateInstance(&createInfo, &instance);
OPENXR_CHECK(result, "Failed to create XrInstance");
```

`OPENXR_CHECK` (from `OpenXRHelper.h`) expands to a check that calls `xrResultToString` and logs a readable error if the call failed. Always use it rather than raw `if (result != XR_SUCCESS)` checks — the error strings are far more useful than the numeric codes.

If this call succeeds, `instance` is your root handle for the rest of the app's lifetime.

---

## Cleanup

When your app exits, destroy the instance:

```cpp
xrDestroyInstance(instance);
instance = XR_NULL_HANDLE;
```

OpenXR follows a strict rule: **child objects must be destroyed before their parent**. The instance is the ultimate parent, so it's always destroyed last. We'll build up the full destruction order across the chapter 02 guides.

---

## Where This Lives in the Tutorial Code

In the Khronos tutorial, instance creation happens in `OpenXRTutorial::CreateInstance()` inside `Chapter2/main.cpp`. The tutorial wraps everything in a class, but the sequence is exactly what's described above:

1. Fill `XrApplicationInfo`
2. Enumerate and select layers + extensions
3. Fill `XrInstanceCreateInfo` (with Android chain on Android)
4. Call `xrCreateInstance`

---

## Quick Reference

| Concept | Key type/call |
|---|---|
| Root handle | `XrInstance` |
| App identity | `XrApplicationInfo` |
| Opt-in capabilities | `enabledExtensionNames[]` |
| Diagnostic layers | `enabledApiLayerNames[]` |
| Android attachment | `XrInstanceCreateInfoAndroidKHR` (chained via `.next`) |
| Create | `xrCreateInstance(&info, &instance)` |
| Destroy | `xrDestroyInstance(instance)` |

---

**Next:** [02.2 — Debug Utils — Your Best Friend](02-debug-utils.md)