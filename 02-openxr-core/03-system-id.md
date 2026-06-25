# 02.3 — System ID and Hardware Queries

> **~10 min read** | Prerequisites: 02.1 (XrInstance)

Before creating a session you need to ask OpenXR: *what hardware is available, and what can it do?* The answer comes back as an `XrSystemId` — a runtime-assigned identifier for the physical XR system attached to the device. This guide covers how to get it and what you can learn from it.

---

## What Is a "System"?

In OpenXR, a **system** is the collection of hardware that makes up an XR experience: the HMD display(s), tracking cameras, controllers, and so on. On a standalone headset like a Meta Quest, the system is the headset itself. On a PC with a connected HMD, the system might be the HMD + base stations.

`XrSystemId` is just a `uint64_t` that the runtime assigns. You never construct one — you query for it by **form factor**.

---

## Form Factors

OpenXR defines two form factors:

| Enum | Meaning |
|---|---|
| `XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY` | Traditional HMD — worn on head, stereoscopic |
| `XR_FORM_FACTOR_HANDHELD_DISPLAY` | Phone-style AR (e.g. ARCore on flat screen) |

For an Android headset app you'll always use `XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY`.

---

## Calling xrGetSystem

```cpp
XrSystemGetInfo systemInfo{XR_TYPE_SYSTEM_GET_INFO};
systemInfo.formFactor = XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY;

XrSystemId systemId = XR_NULL_SYSTEM_ID;
OPENXR_CHECK(xrGetSystem(instance, &systemInfo, &systemId),
    "Failed to get XrSystemId");
```

**This call can fail.** Common reasons:
- No HMD is connected (PC runtime with no headset plugged in)
- The runtime doesn't support the requested form factor
- The runtime isn't running yet

In the Khronos tutorial the code retries this in a loop until the system becomes available, which is correct behavior for production apps.

---

## XrSystemProperties

Once you have a `systemId`, query its properties:

```cpp
XrSystemProperties props{XR_TYPE_SYSTEM_PROPERTIES};
OPENXR_CHECK(xrGetSystemProperties(instance, systemId, &props),
    "Failed to get system properties");
```

The resulting struct gives you:

```cpp
props.systemId;                          // same systemId you passed in
props.vendorId;                          // vendor PCI-style ID
props.systemName;                        // e.g. "Meta Quest 3"
props.graphicsProperties.maxSwapchainImageHeight;
props.graphicsProperties.maxSwapchainImageWidth;
props.graphicsProperties.maxLayerCount;  // max composition layers
props.trackingProperties.orientationTracking; // XrBool32
props.trackingProperties.positionTracking;    // XrBool32
```

The most important fields to check:

- **`positionTracking`** — if this is `XR_FALSE`, you're on a 3DoF device (rotation only). Your app needs to handle this gracefully if it targets multiple platforms.
- **`maxLayerCount`** — the runtime won't accept more composition layers than this in `xrEndFrame`. Always respect it.
- **`maxSwapchainImageWidth/Height`** — useful for validating your swapchain create info later.

---

## Chaining Extension Structs onto Properties

If you're using extensions that add system-level capabilities, you chain their query structs onto `XrSystemProperties.next`:

```cpp
XrSystemHandTrackingPropertiesEXT handProps{
    XR_TYPE_SYSTEM_HAND_TRACKING_PROPERTIES_EXT};

XrSystemProperties props{XR_TYPE_SYSTEM_PROPERTIES};
props.next = &handProps;

xrGetSystemProperties(instance, systemId, &props);

if (handProps.supportsHandTracking) {
    // can use XR_EXT_hand_tracking
}
```

This `next`-chain pattern is consistent across all OpenXR property queries. You'll see it again in chapter 05 when using extensions.

---

## Vulkan Requirements Query

Here's a system query that's specific to the Vulkan graphics binding — and you *must* do it before creating your `VkInstance` and `VkDevice`. The OpenXR runtime has requirements for which Vulkan version and which extensions it needs:

```cpp
PFN_xrGetVulkanGraphicsRequirements2KHR xrGetVulkanGraphicsRequirements2KHR = nullptr;
xrGetInstanceProcAddr(instance,
    "xrGetVulkanGraphicsRequirements2KHR",
    (PFN_xrVoidFunction*)&xrGetVulkanGraphicsRequirements2KHR);

XrGraphicsRequirementsVulkan2KHR vulkanReqs{
    XR_TYPE_GRAPHICS_REQUIREMENTS_VULKAN2_KHR};
OPENXR_CHECK(xrGetVulkanGraphicsRequirements2KHR(instance, systemId, &vulkanReqs),
    "Failed to get Vulkan graphics requirements");

// vulkanReqs.minApiVersionSupported  — minimum Vulkan version required
// vulkanReqs.maxApiVersionSupported  — maximum Vulkan version tested
```

If you skip this call, the runtime may reject your session creation later with a cryptic error. The Khronos tutorial calls this in their `GraphicsAPI_Vulkan` initialization, so if you're using the tutorial's structure it's handled there.

---

## What systemId Is (and Isn't)

`XrSystemId` is **not** a handle you create or destroy. It's an opaque identifier you receive and pass to other calls:

- `xrGetSystemProperties(instance, systemId, ...)`
- `xrCreateSession(instance, &createInfo, ...)` — `createInfo.systemId = systemId`
- `xrEnumerateViewConfigurations(instance, systemId, ...)`
- `xrGetVulkanGraphicsRequirements2KHR(instance, systemId, ...)`

You don't destroy it. It's valid for the lifetime of the instance.

---

## Destruction Order So Far

```
xrDestroyDebugUtilsMessengerEXT(messenger)   // 02.2
xrDestroyInstance(instance)                  // 02.1
```

System IDs don't appear here — they're runtime-managed and disappear when the instance is destroyed.

---

## Quick Reference

| Concept | Key type/call |
|---|---|
| Hardware identifier | `XrSystemId` |
| Request by form factor | `xrGetSystem(instance, &systemInfo, &systemId)` |
| Query capabilities | `xrGetSystemProperties(instance, systemId, &props)` |
| Vulkan version check | `xrGetVulkanGraphicsRequirements2KHR(instance, systemId, &reqs)` |
| Position tracking check | `props.trackingProperties.positionTracking` |

---

**Next:** [02.4 — Creating an XrSession](04-xr-session.md)