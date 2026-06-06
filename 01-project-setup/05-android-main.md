# 01.5 — android_main and the App Entry Point

**Read time:** ~10 minutes  
**Prev:** [Common Helper Files →](04-common-helpers.md)

---

On Android, your entry point isn't `main()` — it's `android_main(struct android_app*)`. This guide explains why and what you need to do inside it before calling any OpenXR functions.

---

## Why Not `main()`?

Android apps start in Java/Kotlin. The OS calls `NativeActivity.onCreate()` on the Java side, which loads your `.so` and calls `ANativeActivity_onCreate()`. The `native_app_glue` NDK helper implements that and spins up a thread that calls your `android_main()`.

So your `android_main()` runs on a dedicated thread, separate from the main Android UI thread.

---

## Step 1: Attach the Thread to the JVM

```cpp
void android_main(struct android_app* app) {
    JNIEnv* env;
    app->activity->vm->AttachCurrentThread(&env, nullptr);
```

OpenXR on Android needs to interact with the JVM (for loader initialization). Before calling any OpenXR function, attach the current thread so JNI calls work.

---

## Step 2: Initialize the OpenXR Loader

This is Android-specific. On other platforms (Windows, Linux), the loader finds the runtime automatically. On Android, you must tell it explicitly where to look:

```cpp
// Get the function pointer (it doesn't exist yet — loader isn't initialized)
PFN_xrInitializeLoaderKHR xrInitializeLoaderKHR = nullptr;
XrInstance dummyInstance = XR_NULL_HANDLE;
OPENXR_CHECK(
    xrGetInstanceProcAddr(XR_NULL_HANDLE, "xrInitializeLoaderKHR",
                          (PFN_xrVoidFunction*)&xrInitializeLoaderKHR),
    "Failed to get xrInitializeLoaderKHR"
);

// Fill in Android context info
XrLoaderInitInfoAndroidKHR loaderInfo{XR_TYPE_LOADER_INIT_INFO_ANDROID_KHR};
loaderInfo.applicationVM      = app->activity->vm;
loaderInfo.applicationContext = app->activity->clazz;

OPENXR_CHECK(
    xrInitializeLoaderKHR((XrLoaderInitInfoBaseHeaderKHR*)&loaderInfo),
    "Failed to initialize loader"
);
```

Why `XR_NULL_HANDLE` for the instance? Because the loader isn't initialized yet — we can still call `xrGetInstanceProcAddr` with a null handle for this one specific function. After `xrInitializeLoaderKHR` succeeds, normal OpenXR initialization can proceed.

---

## Step 3: Set Up Android App State

```cpp
app->userData = &OpenXRTutorial::androidAppState;
app->onAppCmd = OpenXRTutorial::AndroidAppHandleCmd;
OpenXRTutorial::androidApp = app;
```

Android sends lifecycle commands to your app (RESUME, PAUSE, WINDOW_CREATED, DESTROY, etc.) via `onAppCmd`. You set a static callback to handle these. The `AndroidAppState` struct tracks:

```cpp
struct AndroidAppState {
    ANativeWindow* nativeWindow = nullptr;
    bool resumed = false;
};
```

These two bits of state are needed to decide whether OpenXR should be running (session should only run when the app is resumed AND has a window).

---

## Step 4: Enter Your App

```cpp
    OpenXRTutorial_Main(VULKAN);
    // (never reaches here until the XR session ends)
}
```

`OpenXRTutorial_Main()` is your own wrapper that creates the `OpenXRTutorial` class and calls `Run()`. This blocks until the user exits.

---

## The `AndroidAppHandleCmd` Callback

This static method responds to Android lifecycle events:

```cpp
static void AndroidAppHandleCmd(struct android_app* app, int32_t cmd) {
    AndroidAppState* state = (AndroidAppState*)app->userData;
    switch (cmd) {
        case APP_CMD_RESUME:  state->resumed = true;            break;
        case APP_CMD_PAUSE:   state->resumed = false;           break;
        case APP_CMD_INIT_WINDOW:  state->nativeWindow = app->window; break;
        case APP_CMD_TERM_WINDOW:  state->nativeWindow = nullptr;     break;
        case APP_CMD_DESTROY: state->nativeWindow = nullptr;    break;
        default: break;
    }
}
```

---

## Polling System Events Each Frame

Inside your main loop, you must pump Android's event queue:

```cpp
void PollSystemEvents() {
    if (androidApp->destroyRequested != 0) {
        m_applicationRunning = false;
        return;
    }
    while (true) {
        struct android_poll_source* source = nullptr;
        int events = 0;
        // Block only if app is not running (saves battery / avoids spin)
        int timeoutMs = (!androidAppState.resumed && !m_sessionRunning
                         && androidApp->destroyRequested == 0) ? -1 : 0;
        if (ALooper_pollOnce(timeoutMs, nullptr, &events, (void**)&source) >= 0) {
            if (source) source->process(androidApp, source);
        } else {
            break;
        }
    }
}
```

The `timeoutMs = -1` trick makes the loop block (sleep) when there's nothing to do, saving battery. When the session is running and you need to render frames, it's `0` (non-blocking).

Call `PollSystemEvents()` at the top of every iteration of your main loop, before checking OpenXR events.

---

## Full Sequence Summary

```
android_main() called by NativeActivity
  ├── AttachCurrentThread (JNI)
  ├── xrInitializeLoaderKHR (Android-specific loader init)
  ├── Set up AndroidAppState callbacks
  └── → OpenXRTutorial::Run()
          ├── CreateInstance()
          ├── CreateSession()
          └── [main loop]
                ├── PollSystemEvents()   ← Android events
                ├── PollXREvents()       ← OpenXR session state
                └── RenderFrame()        ← if session is running
```

---

📝 Ready to practice? → [Exercise 01: Build a Stub App That Launches](../exercises/01-build-stub-app.md)