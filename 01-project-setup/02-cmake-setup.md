# 01.3 — The Android Manifest for XR

**Read time:** ~6 minutes  
**Prev:** [CMake Setup →](02-cmake-setup.md) | **Next:** [Common Helper Files →](04-common-helpers.md)

---

The `AndroidManifest.xml` tells Android what kind of app this is. For XR, several non-obvious things need to be set.

---

## Key Entries for OpenXR on Android

### Hardware Features

```xml
<uses-feature android:name="android.hardware.vr.headtracking"
              android:required="false" android:version="1" />
<uses-feature android:name="android.hardware.vulkan.version"
              android:required="true" android:version="0x400003" />
<uses-feature android:name="android.hardware.opengles.version"
              android:required="true" android:glEsVersion="0x00030002" />
```

`headtracking required=false` makes your app compatible with both 3DOF and 6DOF devices, and also with devices that aren't standalone all-in-ones. It's counterintuitively the more compatible option.

### The Intent Filter — The Critical Part

This is what tells the OS "this app wants to take over rendering as an immersive XR experience":

```xml
<!-- Standard Khronos/OpenXR way -->
<category android:name="org.khronos.openxr.intent.category.IMMERSIVE_HMD" />

<!-- Meta Quest-specific (still needed for Quest devices) -->
<category android:name="com.oculus.intent.category.VR" />
```

In practice for Quest development, you need **both**. The standard category is the future; the Meta-specific one is what actually works on Quest hardware today.

### NativeActivity

```xml
<activity android:name="android.app.NativeActivity"
          android:configChanges="density|keyboard|keyboardHidden|navigation|orientation|screenLayout|screenSize|uiMode"
          android:exported="true"
          android:screenOrientation="landscape">
    <meta-data android:name="android.app.lib_name"
               android:value="OpenXRTutorialChapter2" />
```

The `lib_name` meta-data tells NativeActivity which `.so` to load. This name must match your CMake `PROJECT_NAME`.

---

## What the `openxr_loader_for_android` Gradle Dependency Does

In `app/build.gradle`:
```groovy
implementation 'org.khronos.openxr:openxr_loader_for_android:1.1.x'
```

This package does two things:
1. Provides OpenXR headers and the compiled `openxr_loader` binaries for Android ABIs
2. Injects its own `AndroidManifest.xml` that merges into yours — setting required package properties for OpenXR to work

You still need your own manifest entries for the intent filters (it doesn't know which device categories you want).

---

**Next:** [Common Helper Files →](04-common-helpers.md)