# 02.4 — Creating an XrSession

> **~10 min read** | Prerequisites: 02.3 (System ID), Vulkan basics

The `XrSession` is the object that ties OpenXR to your graphics API. Once you have one, you can render frames. Getting here requires that your Vulkan device is already initialized — the session and the graphics context are created together, because the runtime needs to know exactly which GPU you're using.

---

## The Chicken-and-Egg Problem (And How OpenXR Solves It)

You might wonder: *how do I create a VkDevice without knowing what OpenXR needs, but how do I know what OpenXR needs without a VkDevice?*

OpenXR uses a two-step handshake:

1. **You query requirements first** — `xrGetVulkanGraphicsRequirements2KHR` (covered in 02.3) tells you the Vulkan version range the runtime supports.
2. **You create Vulkan resources** — using those requirements to guide `VkInstance` / `VkDevice` creation.
3. **You pass those Vulkan resources into session creation** — via `XrGraphicsBindingVulkan2KHR`.

Alternatively, with `XR_KHR_vulkan_enable2`, the runtime can *create* the VkInstance and VkDevice for you and hand them back — which is what the Khronos tutorial does to keep things simple. Both approaches are valid.

---

## The Graphics Binding Struct

The graphics binding tells OpenXR which Vulkan objects to render into. It gets chained onto `XrSessionCreateInfo.next`:

```cpp
XrGraphicsBindingVulkan2KHR graphicsBinding{XR_TYPE_GRAPHICS_BINDING_VULKAN2_KHR};
graphicsBinding.instance       = vkInstance;
graphicsBinding.physicalDevice = vkPhysicalDevice;
graphicsBinding.device         = vkDevice;
graphicsBinding.queueFamilyIndex = graphicsQueueFamilyIndex;
graphicsBinding.queueIndex       = 0;
```

All five fields are required and must be non-null. If any are missing or wrong, the session creation call will fail (and your debug messenger from 02.2 will tell you exactly which field is the problem).

---

## XrSessionCreateInfo

```cpp
XrSessionCreateInfo sessionInfo{XR_TYPE_SESSION_CREATE_INFO};
sessionInfo.next     = &graphicsBinding;  // chain the Vulkan binding
sessionInfo.systemId = systemId;          // from xrGetSystem (02.3)
sessionInfo.createFlags = 0;              // reserved, always 0
```

Then create the session:

```cpp
XrSession session = XR_NULL_HANDLE;
OPENXR_CHECK(xrCreateSession(instance, &sessionInfo, &session),
    "Failed to create XrSession");
```

---

## What Happens Internally

When `xrCreateSession` succeeds, the runtime:

- Associates your Vulkan device with the XR display pipeline
- Allocates internal resources (compositor state, tracking state)
- Sets the session into the `XR_SESSION_STATE_IDLE` state

The session starts in `IDLE`, which means it's created but not yet rendering. You don't render until the runtime tells you to — which is what the event loop (02.5) manages.

---

## Session Lifecycle Overview

It's worth seeing the big picture now, even though 02.5 covers it in depth:

```
                  ┌─────────────────────────────────┐
                  │         XrSession States          │
                  └─────────────────────────────────┘

  xrCreateSession()
        │
        ▼
    UNKNOWN ──► IDLE ──► READY ──► SYNCHRONIZED ──► VISIBLE ──► FOCUSED
                              ▲                                      │
                              └──────────────────────────────────────┘
                                         (back to IDLE on loss)
                  
  xrDestroySession()  ──► LOSS_PENDING ──► EXITING
```

- **IDLE**: session exists, runtime isn't rendering yet
- **READY**: runtime wants you to call `xrBeginSession`
- **SYNCHRONIZED**: you're submitting frames but they may not be visible (e.g. system overlay is on top)
- **VISIBLE**: your frames are visible but you don't have input focus
- **FOCUSED**: full active state — you have both rendering and input focus

Your app should handle all of these transitions. The runtime drives them; your job is to react.

---

## xrBeginSession

When the event loop delivers `XR_SESSION_STATE_READY`, call:

```cpp
XrSessionBeginInfo beginInfo{XR_TYPE_SESSION_BEGIN_INFO};
beginInfo.primaryViewConfigurationType =
    XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO;

OPENXR_CHECK(xrBeginSession(session, &beginInfo), "Failed to begin XrSession");
```

`primaryViewConfigurationType` tells the runtime how you intend to render. For a stereoscopic headset this is always `PRIMARY_STEREO`. Chapter 03 covers view configurations in detail.

---

## xrEndSession

When the runtime signals you should stop (e.g. `XR_SESSION_STATE_STOPPING`), call:

```cpp
OPENXR_CHECK(xrEndSession(session), "Failed to end XrSession");
```

This is not the same as destroying the session. The session still exists after `xrEndSession` — the runtime may later ask you to begin again (e.g. if the user puts the headset back on). Only call `xrDestroySession` when you're fully done.

---

## Destruction Order So Far

```
xrDestroySession(session)                    // new
xrDestroyDebugUtilsMessengerEXT(messenger)   // 02.2
xrDestroyInstance(instance)                  // 02.1
// ... VkDevice, VkInstance destroyed after XR is done
```

> Destroy the session before the Vulkan device — the runtime may still hold references to Vulkan objects until session destruction.

---

## Quick Reference

| Concept | Key type/call |
|---|---|
| Session handle | `XrSession` |
| Vulkan binding | `XrGraphicsBindingVulkan2KHR` (chained on `.next`) |
| Create | `xrCreateSession(instance, &info, &session)` |
| Start rendering | `xrBeginSession(session, &beginInfo)` |
| Stop rendering | `xrEndSession(session)` |
| Destroy | `xrDestroySession(session)` |
| Initial state | `XR_SESSION_STATE_IDLE` |

---

**Next:** [02.5 — The Event Loop & Session State Machine](05-event-loop.md)