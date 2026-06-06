# 00.1 — What is OpenXR and Why It Matters

**Read time:** ~8 minutes  
**Next:** [Core Concepts & Vocabulary →](02-core-concepts.md)

---

## The Problem OpenXR Solves

Before OpenXR, the XR hardware landscape was a mess of proprietary APIs. Oculus had its SDK. HTC Vive had OpenVR. Microsoft had Windows Mixed Reality. Sony had its own stack. If you wanted your app to run on two headsets, you basically had to write two apps.

OpenXR is the Khronos Group's answer to this: a **single, royalty-free, open API** that works across runtimes and hardware — much like how OpenGL and Vulkan unified the graphics API world.

```
Before OpenXR:          After OpenXR:

App                     App
 ├── Oculus SDK          └── OpenXR API
 ├── OpenVR                   ├── Meta Quest Runtime
 ├── WMR SDK                  ├── SteamVR Runtime
 └── ...                      ├── Windows MR Runtime
                              └── ...
```

The promise: **write once, run on any OpenXR-compliant runtime.**

In practice there are still some quirks (vendor extensions, intent filter differences on Android), but the core rendering and input loop is genuinely portable.

---

## What OpenXR Actually Covers

OpenXR is an **XR platform API**, not a graphics API. It handles:

- **Session management** — starting, pausing, ending your XR experience
- **Head tracking** — where the user's head is in 3D space
- **Controller/hand input** — through an abstracted action system
- **Rendering coordination** — telling your app *when* and *where* to render, and handing back images to the compositor
- **Extensions** — vendor-specific features (passthrough, hand tracking, eye tracking, etc.)

What OpenXR does **not** handle:
- How you actually draw pixels (that's Vulkan/OpenGL/D3D's job)
- Audio
- Networking
- Scene graphs or game logic

---

## The Khronos Specification

The authoritative reference is the [OpenXR 1.1 Core Specification](https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html). It's long and dense, but it's also precise — any ambiguity in a tutorial gets resolved there. Bookmark it.

Version 1.0 was released in 2019. As of 2024 the stable version is 1.1. For Android development you'll mostly be on 1.0-compatible runtimes, but the APIs are backward compatible.

---

## Who Implements OpenXR?

A **runtime** is a concrete implementation of the OpenXR API. Examples:
- **Meta Quest runtime** — built into Quest OS
- **SteamVR** — implements OpenXR on PC, works with most PC headsets
- **Windows Mixed Reality** — Microsoft's runtime (being deprecated, but still in the wild)
- **Monado** — the open-source reference runtime for Linux

When you build an OpenXR app for Android (like Quest), the runtime is baked into the device OS. You don't ship it — you link against it through the **loader**.

---

## Key Takeaway

> OpenXR is the USB-C of XR APIs: it standardizes the connector so you can plug into any device.

You're trading engine-level convenience for platform-level control. That control is why you're here.

---

**Next:** [Core Concepts & Vocabulary →](02-core-concepts.md)