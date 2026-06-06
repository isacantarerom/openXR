# 00.3 — How OpenXR Differs from Game Engine XR

**Read time:** ~8 minutes  
**Prev:** [Core Concepts & Vocabulary →](02-core-concepts.md)

---

If you're coming from StereoKit or Unity's XR Toolkit, raw OpenXR will feel like descending several floors. This guide maps what you already know to what you're about to write.

---

## What the Engine Was Hiding

When you called `sk.Run()` in StereoKit or hit Play in Unity, the engine was:

1. Creating an `XrInstance` and `XrSession` for you
2. Managing the session state machine (IDLE → FOCUSED and back)
3. Creating a swapchain per eye, acquiring images each frame, submitting them
4. Calling `xrWaitFrame` / `xrBeginFrame` / `xrEndFrame` in the right order
5. Running `xrSyncActions` every frame for you
6. Translating controller inputs from OpenXR actions into its own input API
7. Handling Android lifecycle events (pause/resume) and propagating them to OpenXR

In raw OpenXR, **you do all of that.** Explicitly. Every frame.

---

## Concept Mapping

| StereoKit / Unity | Raw OpenXR |
|---|---|
| `SK.Initialize()` / `XRSettings.enabled` | `xrCreateInstance()` + `xrCreateSession()` |
| `SK.Run(mainLoop)` | Your own loop calling `xrPollEvent` + `xrWaitFrame` + `xrEndFrame` |
| `Input.Hand(Handed.Right).wrist.Pose` | `xrLocateSpace(handSpace, referenceSpace, ...)` |
| `Input.Controller(Handed.Right).trigger` | `xrGetActionStateFloat(session, &getInfo, &state)` |
| `Renderer.Add(mesh, material, transform)` | Your Vulkan draw calls inside the swapchain acquire/release bracket |
| `Hierarchy.Push(transform)` | Manual matrix math using `XrPosef` |
| Layer system (opaque/additive) | `XrCompositionLayerProjection` submitted in `xrEndFrame` |
| `World.RefreshType == RefreshType.Area` | Choosing `XR_REFERENCE_SPACE_TYPE_STAGE` |

---

## The Three Biggest Mindset Shifts

### 1. You own the session state machine
The OpenXR session is not always running. It transitions through states:

```
IDLE → READY → SYNCHRONIZED → VISIBLE → FOCUSED → VISIBLE → SYNCHRONIZED → ...
```

You must poll for state change events every frame and respond to them (begin the session when READY, stop rendering when VISIBLE, etc.). The engine handled this transparently. You won't.

### 2. You own swapchain management
In StereoKit you never think about swapchains. In raw OpenXR:

- You enumerate supported formats and choose one
- You create one `XrSwapchain` per view (typically 2 for stereo)
- Each frame you call `xrAcquireSwapchainImage`, render into it, then `xrReleaseSwapchainImage`
- The runtime's compositor assembles your images and displays them

This is the most technically dense part of the API. Chapter 03 covers it in detail.

### 3. Input is declarative, not polling
StereoKit gives you imperative input: "read the trigger value right now." OpenXR's action system is declarative: you define what inputs your app *cares about*, suggest bindings to hardware, then sync and read states. It's more setup up front, but the payoff is controller portability.

---

## What You Gain

- **Full control over the render loop** — critical for hitting 72/90/120 Hz targets
- **Access to all extensions** — passthrough, hand mesh, eye tracking, etc.
- **No engine overhead** — no C# GC, no unnecessary scene graph traversal
- **True portability** — the same C++ core can target Quest, PC, Linux (Monado) with minimal changes
- **Deeper understanding** — knowing what the engine does makes you better at using engines too

---

## A Note on Vulkan Specifically

StereoKit uses OpenGL ES internally on Android. Vulkan is more explicit — you control render passes, image layouts, synchronization. OpenXR doesn't care about most of this, but it does:

- Dictate **image formats** for swapchain images (you choose from an enumerated list)
- Expect images to be in a specific **Vulkan image layout** when you release them back to the compositor (`VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` → the runtime transitions from there)
- Require Vulkan **device and queue family selection** to be done in a specific way (queried through OpenXR extensions, not just picked freely)

Chapter 03 covers this Vulkan-OpenXR handshake in detail.

---

**Up next:** Start the actual build — [01.1 Android Studio + NDK Setup →](../01-project-setup/01-android-studio-ndk.md)