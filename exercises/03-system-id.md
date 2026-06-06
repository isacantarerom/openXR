# 02.3 — System ID and Hardware Queries

**Read time:** ~6 minutes  
**Prev:** [Debug Utils →](02-debug-utils.md) | **Next:** [XrSession →](04-xr-session.md)

---

After creating an `XrInstance`, you need to identify the XR hardware you'll be using. That's the **System ID** — an integer handle representing the physical device (headset + runtime combo).

---

## Getting the System ID

```cpp
XrSystemGetInfo systemInfo{XR_TYPE_SYSTEM_GET_INFO};
systemInfo.formFactor = XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY;

XrSystemId systemId = XR_NULL_SYSTEM_ID;
OPENXR_CHECK(xrGetSystem(instance, &systemInfo, &systemId), "Failed to get XrSystemId");
```

`XrFormFactor` has two values:
- `XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY` — headsets (use this for Quest)
- `XR_FORM_FACTOR_HANDHELD_DISPLAY` — handheld AR devices (phone AR)

`xrGetSystem` will fail with `XR_ERROR_FORM_FACTOR_UNAVAILABLE` if no headset is connected. On Android, the runtime is always present (it's part of the OS), so this usually succeeds as long as you're running on-device.

---

## System Properties

Once you have a system ID, you can query what the hardware supports:

```cpp
XrSystemProperties props{XR_TYPE_SYSTEM_PROPERTIES};
OPENXR_CHECK(xrGetSystemProperties(instance, systemId, &props), "Failed to get system properties");

// props.systemName          — e.g. "Oculus Quest2"
// props.vendorId            — numeric vendor ID
// props.graphicsProperties  — max swapchain size, layers
// props.trackingProperties  — orientation/position tracking support
```

`trackingProperties.positionTracking` (bool) tells you whether 6DOF is available. Useful for adapting your locomotion system.

---

## Why System ID Is Not an Object

Unlike `XrInstance` or `XrSession`, `XrSystemId` is just an integer — it has no `xrDestroy*` function. It becomes invalid if the instance is destroyed, but you don't manage its lifecycle explicitly.

The system ID is used when creating an `XrSession` to bind the session to specific hardware.

---

**Next:** [Creating an XrSession →](04-xr-session.md)