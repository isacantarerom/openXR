# 02.2 — Debug Utils — Your Best Friend

> **~10 min read** | Prerequisites: 02.1 (XrInstance)

OpenXR's `XR_EXT_debug_utils` extension gives you a runtime-level message callback — similar to Vulkan's `VK_EXT_debug_utils`. When something goes wrong deep inside the runtime, this is how you find out. Set it up right after instance creation and never develop without it.

---

## Why You Need This

Without debug utils, a bad API call just returns an `XrResult` error code like `XR_ERROR_VALIDATION_FAILURE`. That tells you something went wrong, but not *what*, *where*, or *which field* was invalid.

With debug utils, the runtime (and any API layers) can emit rich messages directly to your callback:

```
[XR_EXT_debug_utils] ERROR (XR_ERROR_VALIDATION_FAILURE):
  xrCreateSession: XrSessionCreateInfo.next chain contains
  XrGraphicsBindingVulkan2KHR with a null VkPhysicalDevice handle.
```

That's actionable. The number alone is not.

---

## Prerequisites

You must have requested the extension at instance creation (covered in 02.1):

```cpp
extensions.push_back(XR_EXT_DEBUG_UTILS_EXTENSION_NAME);
```

And you need to load the extension function pointers after creating the instance — debug utils functions are not statically linked:

```cpp
PFN_xrCreateDebugUtilsMessengerEXT  xrCreateDebugUtilsMessengerEXT  = nullptr;
PFN_xrDestroyDebugUtilsMessengerEXT xrDestroyDebugUtilsMessengerEXT = nullptr;

OPENXR_CHECK(xrGetInstanceProcAddr(instance,
    "xrCreateDebugUtilsMessengerEXT",
    (PFN_xrVoidFunction*)&xrCreateDebugUtilsMessengerEXT),
    "Failed to load xrCreateDebugUtilsMessengerEXT");

OPENXR_CHECK(xrGetInstanceProcAddr(instance,
    "xrDestroyDebugUtilsMessengerEXT",
    (PFN_xrVoidFunction*)&xrDestroyDebugUtilsMessengerEXT),
    "Failed to load xrDestroyDebugUtilsMessengerEXT");
```

> `xrGetInstanceProcAddr` is the OpenXR equivalent of `vkGetInstanceProcAddr`. It's the standard way to retrieve extension function pointers. You'll use this pattern again in chapter 05.

---

## Writing the Callback

Your callback must match `PFN_xrDebugUtilsMessengerCallbackEXT`:

```cpp
static XrBool32 DebugCallback(
    XrDebugUtilsMessageSeverityFlagsEXT severity,
    XrDebugUtilsMessageTypeFlagsEXT     types,
    const XrDebugUtilsMessengerCallbackDataEXT* data,
    void* userData)
{
    // Severity prefix
    const char* severityStr = "UNKNOWN";
    if (severity & XR_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT)   severityStr = "ERROR";
    else if (severity & XR_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) severityStr = "WARN";
    else if (severity & XR_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT)    severityStr = "INFO";
    else if (severity & XR_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT) severityStr = "VERBOSE";

    ALOGD("[OpenXR %s] %s: %s",
        severityStr,
        data->messageId,
        data->message);

    // Return XR_FALSE to not abort the calling function.
    // Return XR_TRUE to abort — only useful for testing error paths.
    return XR_FALSE;
}
```

Key fields in `XrDebugUtilsMessengerCallbackDataEXT`:
- **`message`** — the human-readable description
- **`messageId`** — a short identifier (e.g. `"VUID-XrSessionCreateInfo-next-unique"`)
- **`objectCount` / `objects`** — OpenXR objects involved (handles + type tags)
- **`sessionLabelCount` / `sessionLabels`** — breadcrumb trail if you're using debug labels

---

## Severity and Type Flags

When creating the messenger you specify which messages you want. During development, catch everything:

```cpp
XrDebugUtilsMessageSeverityFlagsEXT severityFlags =
    XR_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT |
    XR_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT    |
    XR_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT |
    XR_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;

XrDebugUtilsMessageTypeFlagsEXT typeFlags =
    XR_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT     |
    XR_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT  |
    XR_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT |
    XR_DEBUG_UTILS_MESSAGE_TYPE_CONFORMANCE_BIT_EXT;
```

For release builds, remove this extension entirely (don't ship the validation overhead).

---

## Creating the Messenger

```cpp
XrDebugUtilsMessengerCreateInfoEXT messengerInfo{
    XR_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT};
messengerInfo.messageSeverities = severityFlags;
messengerInfo.messageTypes      = typeFlags;
messengerInfo.userCallback      = DebugCallback;
messengerInfo.userData          = nullptr;   // pass 'this' if inside a class

XrDebugUtilsMessengerEXT messenger = XR_NULL_HANDLE;
OPENXR_CHECK(xrCreateDebugUtilsMessengerEXT(instance, &messengerInfo, &messenger),
    "Failed to create debug messenger");
```

The messenger is tied to the instance. Destroy it before the instance:

```cpp
xrDestroyDebugUtilsMessengerEXT(messenger);
```

---

## Debug Labels (Optional but Useful)

You can annotate regions of your frame with labels that appear in debug output:

```cpp
XrDebugUtilsLabelEXT label{XR_TYPE_DEBUG_UTILS_LABEL_EXT};
label.labelName = "RenderLoop";

PFN_xrSessionBeginDebugUtilsLabelRegionEXT xrSessionBeginDebugUtilsLabelRegionEXT = ...;
xrSessionBeginDebugUtilsLabelRegionEXT(session, &label);

// ... your render work ...

PFN_xrSessionEndDebugUtilsLabelRegionEXT xrSessionEndDebugUtilsLabelRegionEXT = ...;
xrSessionEndDebugUtilsLabelRegionEXT(session);
```

When a validation error fires inside that region, the callback's `sessionLabels` array will include `"RenderLoop"`, giving you a breadcrumb. This is optional but pays off fast when debugging complex render loops.

---

## Destruction Order So Far

We're building up a teardown sequence. After this guide it looks like:

```
xrDestroyDebugUtilsMessengerEXT(messenger)
xrDestroyInstance(instance)
```

Each subsequent guide adds one more entry above `xrDestroyInstance`.

---

## Quick Reference

| Concept | Key type/call |
|---|---|
| Extension name | `XR_EXT_DEBUG_UTILS_EXTENSION_NAME` |
| Load function pointers | `xrGetInstanceProcAddr(instance, "xrCreate...", ...)` |
| Callback type | `PFN_xrDebugUtilsMessengerCallbackEXT` |
| Messenger handle | `XrDebugUtilsMessengerEXT` |
| Create | `xrCreateDebugUtilsMessengerEXT(instance, &info, &messenger)` |
| Destroy | `xrDestroyDebugUtilsMessengerEXT(messenger)` |

---

**Next:** [02.3 — System ID and Hardware Queries](03-system-id.md)