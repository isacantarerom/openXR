# 02-E — Exercise: Instance + Session + Event Loop

> **Goal:** Wire up the full Chapter 2 lifecycle — instance, debug messenger, system query, session, and event loop — and confirm the app reaches `XR_SESSION_STATE_FOCUSED` without errors.

---

## What You're Building

A minimal Android app that:

1. Creates an `XrInstance` with the Vulkan + debug utils extensions
2. Sets up a `XrDebugUtilsMessengerEXT` and confirms it emits messages
3. Queries the `XrSystemId` and logs system properties
4. Creates an `XrSession` backed by Vulkan
5. Runs the event loop and correctly responds to all session state transitions
6. Reaches `FOCUSED` state — the point where a real app would render frames

You won't render anything yet. The render loop comes in Chapter 03.

---

## Setup

Open the **Chapter2** Android Studio project from your local clone of the Khronos tutorial repo:

```
OpenXR-Tutorials/
└── Chapter2/
    └── app/
        └── build.gradle   ← open this project in Android Studio
```

If you haven't cloned the repo yet, revisit **01.1** and **01.4** — make sure `Common/` is present and the project syncs cleanly before starting this exercise.

---

## Tasks

Work through these in order. Each one builds on the last.

---

### Task 1 — Confirm Debug Messenger is Active

In `OpenXRTutorial::CreateInstance()` (inside `Chapter2/main.cpp`), locate where `xrCreateDebugUtilsMessengerEXT` is called.

**To test it's working:**

Add a deliberate bad call right after instance creation — something that will trigger a validation error. For example, call `xrCreateSession` with a null `systemId`:

```cpp
// Intentional bad call to verify the debug messenger fires
XrSessionCreateInfo badInfo{XR_TYPE_SESSION_CREATE_INFO};
badInfo.systemId = XR_NULL_SYSTEM_ID; // invalid
XrSession dummy = XR_NULL_HANDLE;
xrCreateSession(instance, &badInfo, &dummy); // expect failure
```

**Expected:** The debug messenger callback should fire with `ERROR` severity before the `OPENXR_CHECK` macro logs the failure.

**Remove this after confirming it works.**

---

### Task 2 — Log System Properties

In `CreateInstance()` or a new `LogSystemProperties()` function, after getting the `systemId`, log the following fields:

```
[SystemInfo] Name:             <systemName>
[SystemInfo] Vendor ID:        <vendorId>
[SystemInfo] Position Tracking: <yes/no>
[SystemInfo] Max Swapchain:    <width> x <height>
[SystemInfo] Max Layer Count:  <maxLayerCount>
```

Use `ALOGD` (Android log debug) or your preferred log macro.

**Verify in Logcat** (filter by your app tag) that these values look correct for your target device.

---

### Task 3 — Trace Every Session State Transition

In the event loop's `HandleSessionStateChange` handler, add a log line for **every** state you receive, not just the ones you act on:

```cpp
const char* stateNames[] = {
    "UNKNOWN", "IDLE", "READY", "SYNCHRONIZED",
    "VISIBLE", "FOCUSED", "STOPPING",
    "LOSS_PENDING", "EXITING"
};
ALOGD("[Session] State → %s", stateNames[(int)sessionState]);
```

Run the app and watch the Logcat. **Expected sequence on a headset:**

```
[Session] State → IDLE
[Session] State → READY       ← call xrBeginSession here
[Session] State → SYNCHRONIZED
[Session] State → VISIBLE
[Session] State → FOCUSED     ← app is now "live"
```

If you see `STOPPING` without reaching `FOCUSED`, something is wrong with the session begin path.

---

### Task 4 — Handle LOSS_PENDING and EXITING

Verify your event loop handles these two states without crashing:

- **LOSS_PENDING**: Set a flag that breaks the render loop. Don't just ignore it — on a real device this happens during runtime updates.
- **EXITING**: Same — break the loop and proceed to teardown.

To test without a real runtime loss, you can temporarily add a timer that forces an early exit from the main loop after 5 seconds and verify cleanup runs without Vulkan validation errors.

---

### Task 5 — Verify the Destruction Order

Add a log line inside each destructor/cleanup call:

```
[Cleanup] Destroying XrSession
[Cleanup] Destroying DebugMessenger
[Cleanup] Destroying XrInstance
[Cleanup] Destroying VkDevice
[Cleanup] Destroying VkInstance
```

Run the app, let it reach `FOCUSED`, then press the back button (or wait for it to exit). **Verify in Logcat that the lines appear in exactly this order.** Any other order risks use-after-free crashes that may not show up immediately.

---

## Stretch Goals

These are optional — do them if you want to go deeper.

**A — Handle REFERENCE_SPACE_CHANGE_PENDING**  
Add a case for `XR_TYPE_EVENT_DATA_REFERENCE_SPACE_CHANGE_PENDING` and log the `changeTime` and `poseValid` fields. Trigger it by recentering the headset (usually a long press on the menu button).

**B — Query Extension Support**  
Before creating the instance, call `xrEnumerateInstanceExtensionProperties` and log all available extensions. Compare the list against what you requested. Note any extensions your target device doesn't support.

**C — Query API Layer Support**  
Call `xrEnumerateApiLayerProperties` and check whether `XR_APILAYER_LUNARG_core_validation` is available. Add fallback logic that skips the layer if it's not present (so the app still runs on devices where the layer isn't installed).

---

## What Success Looks Like

In Logcat you should see, in order:

```
[OpenXR INFO]  Instance created (runtime: <runtime name>)
[OpenXR VERBOSE] ... (debug messenger noise — this is good)
[SystemInfo] Name: <device>
[SystemInfo] Position Tracking: yes
[Session] State → IDLE
[Session] State → READY
[Session] State → SYNCHRONIZED
[Session] State → FOCUSED
```

And on exit:

```
[Cleanup] Destroying XrSession
[Cleanup] Destroying DebugMessenger
[Cleanup] Destroying XrInstance
```

No validation errors, no native crashes, no skipped states.

---

## Common Mistakes

| Symptom | Likely cause |
|---|---|
| `xrCreateSession` fails with `XR_ERROR_VALIDATION_FAILURE` | Vulkan binding struct has a null handle |
| App stuck in `IDLE` forever | `xrBeginSession` not called when state reaches `READY` |
| Crash on exit | Wrong destruction order (session after instance, or Vulkan before XR) |
| No debug messenger output | Extension not requested, or function pointer not loaded |
| `xrGetSystem` returns `XR_ERROR_FORM_FACTOR_UNAVAILABLE` | Device/emulator not recognized as an HMD |

---

**Next:** [03.1 — View Configurations & Stereo Setup](../03-graphics/01-view-configurations.md)