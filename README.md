# 🥽 OpenXR for Android + Vulkan — Self-Guided Study

> A bite-sized, topic-by-topic guide to raw OpenXR development on Android using Vulkan.  
> Each guide is designed to take **~10 minutes** to read. Exercises follow where hands-on practice matters.

This curriculum covers everything in the [official Khronos OpenXR tutorial](https://openxr-tutorial.com/android/vulkan/1-introduction.html) and then some — reorganized into digestible chunks you can actually enjoy reading.

---

## 📋 Syllabus

### 🧠 00 — Fundamentals
> *Why OpenXR exists, and the vocabulary you need before writing a single line.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 00.1 | [What is OpenXR and Why It Matters](00-fundamentals/01-what-is-openxr.md) | XR fragmentation problem, OpenXR's role, Khronos ecosystem |
| 00.2 | [Core Concepts & Vocabulary](00-fundamentals/02-core-concepts.md) | Instance, Session, Runtime, Loader, API Layers, Actions, Poses |
| 00.3 | [How OpenXR Differs from Game Engine XR](00-fundamentals/03-openxr-vs-engines.md) | StereoKit/Unity vs raw OpenXR — what abstractions you're now owning yourself |

---

### 🛠️ 01 — Project Setup (Android + Vulkan)
> *Getting a real build environment going. The boring-but-necessary stuff, made fast.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 01.1 | [Android Studio + NDK Setup](01-project-setup/01-android-studio-ndk.md) | Installing Android Studio, NDK, Vulkan support, USB debugging |
| 01.2 | [CMake for Native Android XR Projects](01-project-setup/02-cmake-setup.md) | CMakeLists.txt structure, FetchContent for OpenXR SDK, native_app_glue |
| 01.3 | [The Android Manifest for XR](01-project-setup/03-android-manifest.md) | Intent filters, `IMMERSIVE_HMD` vs `com.oculus.intent`, Vulkan feature flags |
| 01.4 | [The Common Helper Files](01-project-setup/04-common-helpers.md) | DebugOutput, OpenXRHelper, GraphicsAPI, OPENXR_CHECK macro |
| 01.5 | [android_main and the App Entry Point](01-project-setup/05-android-main.md) | NativeActivity, JVM attachment, loader initialization, AndroidAppState |

📝 **Exercise:** [01-E — Build a Stub App That Launches](exercises/01-build-stub-app.md)

---

### ⚙️ 02 — OpenXR Core Setup
> *The lifecycle of an OpenXR application. The session state machine is the heart of everything.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 02.1 | [Creating an XrInstance](02-openxr-core/01-xr-instance.md) | XrApplicationInfo, API layers, instance extensions, xrCreateInstance |
| 02.2 | [Debug Utils — Your Best Friend](02-openxr-core/02-debug-utils.md) | XR_EXT_debug_utils, XrDebugUtilsMessengerEXT, callback setup |
| 02.3 | [System ID and Hardware Queries](02-openxr-core/03-system-id.md) | xrGetSystem, XrFormFactor, XrSystemProperties |
| 02.4 | [Creating an XrSession](02-openxr-core/04-xr-session.md) | Binding the graphics API, XrSessionCreateInfo, session lifecycle overview |
| 02.5 | [The Event Loop & Session State Machine](02-openxr-core/05-event-loop.md) | xrPollEvent, XrSessionState transitions, xrBeginSession / xrEndSession |

📝 **Exercise:** [02-E — Instance + Session + Event Loop](exercises/02-instance-session-eventloop.md)

---

### 🎨 03 — Graphics (Swapchains & Rendering)
> *Where OpenXR hands off to Vulkan. Swapchains are the key concept to internalize here.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 03.1 | [View Configurations & Stereo Setup](03-graphics/01-view-configurations.md) | XrViewConfigurationType, PRIMARY_STEREO, enumerating views |
| 03.2 | [Swapchain Concepts](03-graphics/02-swapchain-concepts.md) | What an XrSwapchain is, why it's different from a raw Vulkan swapchain |
| 03.3 | [Creating Swapchains](03-graphics/03-creating-swapchains.md) | Format enumeration, XrSwapchainCreateInfo, image acquisition |
| 03.4 | [Reference Spaces & Coordinate Systems](03-graphics/04-reference-spaces.md) | LOCAL, STAGE, VIEW spaces, XrPosef, coordinate conventions |
| 03.5 | [The Render Loop](03-graphics/05-render-loop.md) | xrWaitFrame, xrBeginFrame, xrEndFrame, XrFrameState |
| 03.6 | [Rendering Layers & Projection Views](03-graphics/06-rendering-layers.md) | XrCompositionLayerProjection, XrCompositionLayerProjectionView, blend modes |
| 03.7 | [Drawing Geometry (Vulkan side)](03-graphics/07-drawing-geometry.md) | Hooking Vulkan renderpass into the swapchain image, depth buffers |

📝 **Exercise:** [03-E — Render a Colored Clearing](exercises/03-render-loop.md)  
📝 **Exercise:** [03-E2 — Draw Cuboids in Space](exercises/03b-cuboids.md)

---

### 🎮 04 — Interactions & Input (Actions System)
> *OpenXR's input model is action-based, not button-based. It's a mindset shift.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 04.1 | [The Actions System — Big Picture](04-interactions/01-actions-overview.md) | Why actions exist, semantic vs physical inputs, bindings |
| 04.2 | [Defining Action Sets and Actions](04-interactions/02-action-sets.md) | xrCreateActionSet, xrCreateAction, action types |
| 04.3 | [Interaction Profiles & Suggested Bindings](04-interactions/03-interaction-profiles.md) | /interaction_profiles/..., xrSuggestInteractionProfileBindings |
| 04.4 | [Attaching Actions to a Session](04-interactions/04-attaching-actions.md) | xrAttachSessionActionSets, xrSyncActions |
| 04.5 | [Reading Action States](04-interactions/05-reading-action-states.md) | xrGetActionStateBoolean/Float/Pose, XrActionStatePose |
| 04.6 | [Spaces from Poses & Hand Tracking](04-interactions/06-pose-spaces.md) | xrCreateActionSpace, locating spaces, grip vs aim poses |
| 04.7 | [Haptic Output](04-interactions/07-haptics.md) | xrApplyHapticFeedback, XrHapticVibration |

📝 **Exercise:** [04-E — Read a Trigger + Move a Cube](exercises/04-actions-input.md)

---

### 🔌 05 — Extensions
> *Where OpenXR gets vendor-specific and powerful.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 05.1 | [How Extensions Work](05-extensions/01-extensions-overview.md) | Extension loading pattern, xrGetInstanceProcAddr, availability checking |
| 05.2 | [Useful Cross-Vendor Extensions](05-extensions/02-common-extensions.md) | XR_KHR_visibility_mask, XR_EXT_hand_tracking, XR_FB_passthrough overview |
| 05.3 | [Vendor-Specific Extensions (Meta / Other)](05-extensions/03-vendor-extensions.md) | Meta's extension catalog, capability negotiation, platform differences |
| 05.4 | [API Layers Revisited](05-extensions/04-api-layers.md) | Validation layer, LunarG tools, custom layer authoring basics |

---

### 🚀 06 — Next Steps & Going Deeper
> *Where to go from here.*

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 06.1 | [Recommended Resources](06-next-steps/01-resources.md) | Khronos spec, hello_xr sample, OpenXR-SDK-Source, community |
| 06.2 | [Performance Considerations for XR](06-next-steps/02-performance.md) | Frame timing, foveated rendering, fixed foveation extension |
| 06.3 | [Porting Between Platforms](06-next-steps/03-porting.md) | Windows Vulkan vs Android Vulkan differences, runtime quirks |
| 06.4 | [Project Ideas to Solidify Your Knowledge](06-next-steps/04-project-ideas.md) | A handful of progressively harder projects to build |

---

## 🗂️ How This Repo Is Organized

```
openxr-android-vulkan-guide/
├── README.md                  ← You are here (syllabus)
├── 00-fundamentals/           ← Concepts & vocabulary
├── 01-project-setup/          ← Build environment guides
├── 02-openxr-core/            ← Instance, Session, Event loop
├── 03-graphics/               ← Swapchains, render loop, geometry
├── 04-interactions/           ← Actions system & input
├── 05-extensions/             ← Extension loading & useful extensions
├── 06-next-steps/             ← Resources, perf, porting
└── exercises/                 ← Hands-on exercise guides
```

---

## 🎯 Prerequisites

You should be comfortable with:
- **C/C++** — pointers, structs, basic memory management
- **Vulkan basics** — you don't need to be an expert, but knowing what a VkDevice and VkImage are will help
- **Android development** — enough to know what an APK is and how to sideload

If you're coming from **StereoKit or Unity XR**, check out [00.3 — How OpenXR Differs from Game Engine XR](00-fundamentals/03-openxr-vs-engines.md) first.

---

## 📚 Primary References

- [Khronos OpenXR Specification 1.1](https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html)
- [OpenXR Tutorial (official Khronos)](https://openxr-tutorial.com/android/vulkan/1-introduction.html)
- [hello_xr sample — OpenXR-SDK-Source](https://github.com/KhronosGroup/OpenXR-SDK-Source/tree/main/src/tests/hello_xr)
- [OpenXR Loader documentation](https://registry.khronos.org/OpenXR/specs/1.1/loader.html)