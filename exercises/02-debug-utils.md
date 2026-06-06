# 02.2 — Debug Utils: Your Best Friend

**Read time:** ~7 minutes  
**Prev:** [XrInstance →](01-xr-instance.md) | **Next:** [System ID →](03-system-id.md)

---

`XR_EXT_debug_utils` gives you a callback that fires whenever OpenXR detects an issue. Enable it during development — it will save you hours.

---

## How It Works

1. You enable the `XR_EXT_DEBUG_UTILS_EXTENSION_NAME` extension when creating your instance
2. You create an `XrDebugUtilsMessengerEXT` object with a callback function
3. The runtime calls your callback when it detects invalid API usage, errors, or warnings

---

## The Callback Function

```cpp
XrBool32 OpenXRMessageCallbackFunction(
    XrDebugUtilsMessageSeverityFlagsEXT severity,
    XrDebugUtilsMessageTypeFlagsEXT type,
    const XrDebugUtilsMessengerCallbackDataEXT* callbackData,
    void* userData)
{
    auto ToString = [](XrDebugUtilsMessageSeverityFlagsEXT s) -> std::string {
        if (s & XR_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT)   return "ERROR";
        if (s & XR_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) return "WARNING";
        if (s & XR_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT)    return "INFO";
        return "VERBOSE";
    };
    std::cerr << "[OpenXR " << ToString(severity) << "] "
              << callbackData->message << std::endl;
    if (severity & XR_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT) {
        DEBUG_BREAK; // Stop execution on errors
    }
    return XR_FALSE; // Don't abort the OpenXR call
}
```

---

## Creating the Messenger

Because `XR_EXT_debug_utils` is an extension, its functions aren't loaded automatically. You must fetch them via `xrGetInstanceProcAddr`. The `OpenXRDebugUtils.cpp` helper does this for you:

```cpp
XrDebugUtilsMessengerEXT debugMessenger = XR_NULL_HANDLE;
CreateOpenXRDebugUtilsMessenger(instance, &debugMessenger);
```

Call this **immediately after** `xrCreateInstance`. Destroy it **just before** `xrDestroyInstance`:

```cpp
DestroyOpenXRDebugUtilsMessenger(instance, debugMessenger);
```

---

## What It Catches

- Calling functions out of order (e.g., creating a session before a system ID is obtained)
- Passing invalid handles or null where non-null is required
- Using extensions that weren't enabled
- Missing `type` fields in structs
- Many more spec violations

Without this, you get cryptic error codes. With it, you get messages like:
> `XrSession is being used but has not yet been created`

---

**Next:** [System ID and Hardware Queries →](03-system-id.md)