# 02.5 — The Event Loop & Session State Machine

**Read time:** ~10 minutes  
**Prev:** [XrSession →](04-xr-session.md)

---

The OpenXR session state machine is the heartbeat of your XR application. Understanding it is essential — every frame, you must poll for events and respond to state transitions correctly.

---

## The State Machine

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
[App Start] → UNKNOWN → IDLE → READY → SYNCHRONIZED → VISIBLE → FOCUSED
                                 │                              │
                                 └──── (user removes headset) ──┘
                                         ↓
                                      STOPPING → IDLE → EXITING
```

| State | Meaning | What to do |
|-------|---------|------------|
| `UNKNOWN` | Initial state, no events yet | Wait |
| `IDLE` | Session created but not active | Wait for READY |
| `READY` | Runtime wants you to begin rendering | Call `xrBeginSession()` |
| `SYNCHRONIZED` | Session running, but not visible | Render at reduced rate or skip |
| `VISIBLE` | Visible but not focused (e.g., system overlay open) | Render, but no input |
| `FOCUSED` | Fully active, input available | Render + process input |
| `STOPPING` | Runtime wants you to stop | Call `xrEndSession()` |
| `EXITING` | Session ending | Destroy session, exit loop |
| `LOSS_PENDING` | Session will be lost (device disconnected) | Handle gracefully |

---

## xrPollEvent

Every frame (or at least once per loop iteration), call:

```cpp
XrEventDataBuffer event{XR_TYPE_EVENT_DATA_BUFFER};
XrResult result = xrPollEvent(instance, &event);

while (result == XR_SUCCESS) {
    switch (event.type) {
        case XR_TYPE_EVENT_DATA_SESSION_STATE_CHANGED: {
            auto* stateChanged = (XrEventDataSessionStateChanged*)&event;
            HandleSessionStateChange(stateChanged->state);
            break;
        }
        case XR_TYPE_EVENT_DATA_INSTANCE_LOSS_PENDING: {
            m_applicationRunning = false;
            break;
        }
        default:
            break;
    }
    event = {XR_TYPE_EVENT_DATA_BUFFER}; // reset before next poll
    result = xrPollEvent(instance, &event);
}
```

`XR_EVENT_UNAVAILABLE` means no more events — that's expected, not an error.

---

## Handling Session State Changes

```cpp
void HandleSessionStateChange(XrSessionState newState) {
    m_sessionState = newState;

    if (newState == XR_SESSION_STATE_READY) {
        XrSessionBeginInfo beginInfo{XR_TYPE_SESSION_BEGIN_INFO};
        beginInfo.primaryViewConfigurationType = XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO;
        OPENXR_CHECK(xrBeginSession(session, &beginInfo), "Failed to begin session");
        m_sessionRunning = true;
    }

    if (newState == XR_SESSION_STATE_STOPPING) {
        OPENXR_CHECK(xrEndSession(session), "Failed to end session");
        m_sessionRunning = false;
    }

    if (newState == XR_SESSION_STATE_EXITING ||
        newState == XR_SESSION_STATE_LOSS_PENDING) {
        m_applicationRunning = false;
    }
}
```

---

## xrBeginSession vs xrEndSession

- `xrBeginSession` — called once when state → READY. Tells the runtime you're ready to render. Must specify the view configuration type (always `PRIMARY_STEREO` for headsets).
- `xrEndSession` — called once when state → STOPPING. Tells the runtime you've stopped rendering. Does not destroy the session.

After `xrEndSession`, the session transitions back toward IDLE and may become READY again (e.g., user re-dons the headset). Your loop must handle this.

---

## The Main Loop Structure

```cpp
void Run() {
    CreateInstance();
    CreateSession();
    // ... other setup ...

    while (m_applicationRunning) {
        PollSystemEvents();  // Android OS events
        PollXREvents();      // OpenXR session state events

        if (m_sessionRunning) {
            RenderFrame();
        }
    }

    // Teardown in reverse order
    DestroySession();
    DestroyInstance();
}
```

---

📝 Ready to practice? → [Exercise 02: Instance + Session + Event Loop](../exercises/02-instance-session-eventloop.md)