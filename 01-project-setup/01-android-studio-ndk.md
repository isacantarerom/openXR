# 01.1 — Android Studio + NDK Setup

**Read time:** ~8 minutes  
**Next:** [CMake for Native Android XR Projects →](02-cmake-setup.md)

---

## What You Need

| Tool | Where to get it | Notes |
|------|----------------|-------|
| Android Studio | [developer.android.com/studio](https://developer.android.com/studio) | Any recent version |
| NDK (Native Dev Kit) | Via Android Studio SDK Manager | Provides C/C++ compilers for Android |
| Android device | Meta Quest 2/3, or any OpenXR-capable Android headset | Minimum API 24 (Android 7.0) |

---

## Installing the NDK

Inside Android Studio:

1. Open **SDK Manager** (top menu or the toolbar icon)
2. Go to **SDK Tools** tab
3. Check **NDK (Side by side)** and **CMake**
4. Click **Apply** → let it download

You want NDK version **r25c** or later. The tutorial uses CMake 3.28.3+.

---

## Enabling USB Debugging on Your Headset

For Meta Quest:
1. Create a Meta developer account (free)
2. In the Quest settings → Developer mode → enable it
3. Connect via USB → accept the prompt on the headset
4. Run `adb devices` in terminal — your device should appear

```bash
adb devices
# List of devices attached
# 1WMHHA123456789    device   ← good
```

If you see `unauthorized`, look for a dialog on the headset asking to allow USB debugging.

---

## Project Structure Overview

The workspace you'll build looks like this:

```
workspace/
├── cmake/
│   └── glsl_shader.cmake        ← shader compilation helper
├── Common/
│   ├── GraphicsAPI.h/.cpp
│   ├── GraphicsAPI_Vulkan.h/.cpp
│   ├── OpenXRHelper.h
│   ├── OpenXRDebugUtils.h/.cpp
│   ├── DebugOutput.h
│   └── HelperFunctions.h
├── Shaders/
│   ├── VertexShader.glsl
│   └── PixelShader.glsl
└── Chapter2/                    ← your app
    ├── CMakeLists.txt
    ├── main.cpp
    └── app/
        ├── build.gradle
        └── src/main/
            └── AndroidManifest.xml
```

The `Common/` folder is shared across chapters. You'll download these helper files from the tutorial — they handle Vulkan/OpenXR plumbing so you can focus on the API concepts.

---

## Creating the Android Studio Project (Quick Path)

The fastest way is to download the prebuilt `AndroidBuildFolder.zip` from the Khronos releases and rename it to `Chapter2`. Then open that folder in Android Studio.

If you want to build it by hand (good learning exercise), the detailed steps are in the [official tutorial ch. 1.4](https://openxr-tutorial.com/android/vulkan/1-introduction.html#cmake-and-project-files) — it covers creating a Native Activity project from scratch and stripping out the Java/Kotlin scaffolding.

**Key settings when creating the project manually:**
- Language: doesn't matter (we're using C++)
- Minimum SDK: **API 24** (Android 7.0 Nougat)
- Build Config Language: **Groovy DSL** (`build.gradle`, not `.kts`)

---

## Verifying Your Setup

After opening the project, do a Gradle sync:  
`File → Sync Project with Gradle Files`

A successful sync with no errors means your NDK and CMake are properly installed. You won't be able to build yet (no source files), but the sync should pass clean.

---

**Next:** [CMake for Native Android XR Projects →](02-cmake-setup.md)