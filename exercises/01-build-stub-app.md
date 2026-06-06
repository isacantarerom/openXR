# 02.1 — Creating an XrInstance

**Read time:** ~10 minutes  
**Next:** [Debug Utils →](02-debug-utils.md)

---

`XrInstance` is the root of everything. You create it first, destroy it last. It encapsulates your app's connection to the OpenXR loader and runtime.

---

## What Goes Into an Instance

Three things:
1. **Application info** — your app name, engine name, OpenXR API version
2. **API layers** — optional middleware (validation, tracing)
3. **Instance extensions** — additional functionality (graphics API binding, debug utils)

---

## XrApplicationInfo

```cpp
XrApplicationInfo appInfo{};
strncpy(appInfo.applicationName, "My XR App", XR_MAX_APPLICATION_NAME_SIZE);
appInfo.applicationVersion = 1;
strncpy(appInfo.engineName, "My Engine", XR_MAX_ENGINE_NAME_SIZE);
appInfo.engineVersion = 1;
appInfo.apiVersion = XR_CURRENT_API_VERSION;
```

Note: the name fields are `char[]` arrays, not pointers — use `strncpy`. `XR_CURRENT_API_VERSION` picks up whatever version of OpenXR your headers are for (1.0 or 1.1).

---

## Choosing Extensions

```cpp
std::vector<const char*> extensions;
extensions.push_back(XR_EXT_DEBUG_UTILS_EXTENSION_NAME);   // validation & logging
extensions.push_back(XR_KHR_VULKAN_ENABLE2_EXTENSION_NAME); // Vulkan graphics binding
```

Not all extensions are available on every runtime. You must check before enabling:

```cpp
uint32_t count = 0;
xrEnumerateInstanceExtensionProperties(nullptr, 0, &count, nullptr);
std::vector<XrExtensionProperties> available(count, {XR_TYPE_EXTENSION_PROPERTIES});
xrEnumerateInstanceExtensionProperties(nullptr, count, &count, available.data());

// Only enable extensions that exist in `available`
std::vector<const char*> activeExtensions;
for (auto& wanted : extensions) {
    for (auto& ext : available) {
        if (strcmp(wanted, ext.extensionName) == 0) {
            activeExtensions.push_back(wanted);
            break;
        }
    }
}
```

The same pattern applies to API layers.

---

## Creating the Instance

```cpp
XrInstanceCreateInfo createInfo{XR_TYPE_INSTANCE_CREATE_INFO};
createInfo.applicationInfo      = appInfo;
createInfo.enabledApiLayerCount = (uint32_t)activeLayers.size();
createInfo.enabledApiLayerNames = activeLayers.data();
createInfo.enabledExtensionCount = (uint32_t)activeExtensions.size();
createInfo.enabledExtensionNames = activeExtensions.data();

XrInstance instance = XR_NULL_HANDLE;
OPENXR_CHECK(xrCreateInstance(&createInfo, &instance), "Failed to create XrInstance");
```

The `{XR_TYPE_...}` pattern is universal in OpenXR. Every struct has a `type` field that identifies it to the runtime — always set it using the struct's designated initializer or by assigning directly.

---

## Checking Instance Properties

```cpp
XrInstanceProperties props{XR_TYPE_INSTANCE_PROPERTIES};
OPENXR_CHECK(xrGetInstanceProperties(instance, &props), "Failed to get instance properties");
// props.runtimeName, props.runtimeVersion — useful for logging
```

Good for logging which runtime you're running on.

---

## Destruction

```cpp
xrDestroyInstance(instance);
```

Must be called after destroying all child objects (session, swapchains, etc.).

---

**Next:** [Debug Utils — Your Best Friend →](02-debug-utils.md)