# 00.2 — Core Concepts & Vocabulary

**Read time:** ~10 minutes  
**Prev:** [What is OpenXR →](01-what-is-openxr.md) | **Next:** [OpenXR vs Game Engines →](03-openxr-vs-engines.md)

---

Before writing any code, you need the vocabulary. OpenXR has a specific meaning for every term below — don't let intuition mislead you.

---

## The Object Hierarchy

OpenXR objects are created in a strict hierarchy. You can't skip steps.

```
XrInstance
  └── XrSystemId          (hardware query — not an object, just an ID)
        └── XrSession
              ├── XrSwapchain(s)
              ├── XrSpace(s)
              └── XrActionSet(s)
                    └── XrAction(s)
```

---

## Key Terms

### XrInstance
The root object. Created first, destroyed last. Represents your app's connection to the OpenXR runtime. Analogous to `VkInstance` in Vulkan. Holds the selected API version, enabled layers, and enabled instance extensions.

### Runtime
The software that implements the OpenXR API. On a Meta Quest, this is baked into the OS. Your app talks to the runtime through the **loader**.

### Loader
A library (`openxr_loader`) that ships with your app. Its job: find the runtime on the device and forward your OpenXR calls to it. On Android, you initialize it explicitly before calling anything else (see `xrInitializeLoaderKHR`).

### API Layers
Optional middleware inserted between your app and the runtime. The **validation layer** (`XrApiLayer_core_validation`) is the most important one during development — it catches incorrect API usage and logs helpful errors. Layers are enabled at `XrInstance` creation time.

### XrSystemId
Not an object — just an integer ID representing the physical XR hardware available on the device (the headset + controllers). You get it by calling `xrGetSystem()` with a form factor (almost always `XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY`).

### XrSession
The core runtime object for your XR experience. Binds OpenXR to a specific graphics API (Vulkan, in our case). Has a **state machine** — the session goes through states like IDLE → READY → SYNCHRONIZED → VISIBLE → FOCUSED as the user puts on/takes off the headset.

### XrSwapchain
OpenXR's mechanism for handing you images to render into. The runtime controls the images; you acquire one, render into it, and release it. This is how the compositor picks up your frames. Each eye typically gets its own swapchain.

### XrSpace
A coordinate frame in the world. Used to locate things (your head, your hands, a virtual object) relative to each other. Common spaces:
- `VIEW` — relative to the head
- `LOCAL` — relative to where the session started
- `STAGE` — relative to the play area (floor-level origin)

### XrPose / XrPosef
A position (`XrVector3f`) plus orientation (`XrQuaternionf`) in 3D space. This is how everything is located — your head, hands, controllers.

### Actions System
OpenXR's input model. Instead of polling "is button A pressed?", you define **actions** with semantic meaning ("grab", "teleport", "menu"). The runtime maps physical controller inputs to your actions based on **interaction profiles**. This makes your input code work across different controllers automatically.

### XrAction
A named, typed input or output: boolean (button), float (trigger), pose (hand position), or vibration (haptic).

### XrActionSet
A logical group of actions. You can have multiple sets (e.g., "locomotion", "UI interaction") and activate/deactivate them per frame.

### Interaction Profile
A description of a specific controller type (e.g., `/interaction_profiles/oculus/touch_controller`). You suggest bindings (which physical input maps to which action) for each profile.

### Binding
The mapping from a physical controller path (like `/user/hand/right/input/trigger/value`) to an `XrAction`.

---

## The OpenXR Coordinate System

OpenXR uses a **right-handed coordinate system**:
- `+X` → right
- `+Y` → up  
- `+Z` → toward the viewer (out of the screen)

Units are **meters**. Always.

---

## XrResult — Error Handling

Almost every OpenXR function returns an `XrResult`. Positive values are success codes; negative values are errors. The `OPENXR_CHECK` macro (from the tutorial's common files) wraps this into an assertion with a useful error message. Use it everywhere while learning.

```cpp
XrResult result = xrSomeFunction(...);
// XR_SUCCESS == 0
// XR_ERROR_* are negative
```

---

## Quick Reference Card

| Term | One-line definition |
|------|---------------------|
| `XrInstance` | Your app's root connection to OpenXR |
| `XrSystemId` | The hardware (headset) you're targeting |
| `XrSession` | The active XR experience, bound to a graphics API |
| `XrSwapchain` | The images you render into (owned by the runtime) |
| `XrSpace` | A coordinate frame for locating things in 3D |
| `XrPosef` | A position + orientation |
| `XrAction` | A named, semantic input/output |
| `XrActionSet` | A group of actions |
| Runtime | The OpenXR implementation on the device |
| Loader | The library that connects your app to the runtime |
| API Layer | Optional middleware (e.g., validation, tracing) |

---

**Next:** [How OpenXR Differs from Game Engine XR →](03-openxr-vs-engines.md)