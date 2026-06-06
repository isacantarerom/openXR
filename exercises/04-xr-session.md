# 02.4 — Creating an XrSession

**Read time:** ~8 minutes  
**Prev:** [System ID →](03-system-id.md) | **Next:** [Event Loop & Session State Machine →](05-event-loop.md)

---

The `XrSession` is the active XR experience. It binds OpenXR to a specific graphics API and represents the ongoing interaction between your app and the runtime's compositor.

---

## The Graphics Binding

Before creating a session, OpenXR needs to know which graphics API and which specific GPU objects to use. For Vulkan, this is done via `XrGraphicsBindingVulkan2KHR`:

```cpp
XrGraphicsBindingVulkan2KHR vulkanBinding{XR_TYPE_GRAPHICS_BINDING_VULKAN2_KHR};
vulkanBinding.instance       = vkInstance;
vulkanBinding.physicalDevice = vkPhysicalDevice;
vulkanBinding.device         = vkDevice;
vulkanBinding.queueFamilyIndex = graphicsQueueFamilyIndex;
vulkanBinding.queueIndex     = 0;
```

**Important:** You can't just pick any `VkPhysicalDevice`. OpenXR specifies which one to use via `xrGetVulkanGraphicsDevice2KHR`. Using a different device will result in failures or undefined behavior. The `GraphicsAPI_Vulkan` helper handles this correctly.

---

## Session Create Info

```cpp
XrSessionCreateInfo sessionInfo{XR_TYPE_SESSION_CREATE_INFO};
sessionInfo.next     = &vulkanBinding;  // chain the graphics binding
sessionInfo.systemId = systemId;

XrSession session = XR_NULL_HANDLE;
OPENXR_CHECK(xrCreateSession(instance, &sessionInfo, &session), "Failed to create XrSession");
```

The `next` pointer chain is a recurring OpenXR pattern — it's how extension-specific data is attached to base structs without modifying them.

---

## Session Lifecycle Overview

A session is not immediately "running" after creation. It goes through a state machine driven by events. See [02.5 — The Event Loop](05-event-loop.md) for the full state machine.

---

## Destruction

```cpp
xrDestroySession(session);
```

Must happen before `xrDestroyInstance`. Destroying the session does not destroy associated action sets — those are instance-level objects.

---

**Next:** [The Event Loop & Session State Machine →](05-event-loop.md)