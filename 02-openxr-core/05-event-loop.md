# 02.5 — The Event Loop & Session State Machine

> **~10 min read** | Prerequisites: 02.4 (XrSession)

The session state machine is the beating heart of an OpenXR application. The runtime is in charge — it decides when your app renders, when it gets input focus, and when it should stop. Your job is to poll for events and respond correctly to each state transition. Get this wrong and nothing else in the app will work.

---

## The Core Idea

OpenXR uses a **pull model** for events: the runtime doesn't call you, you call it. Once per frame (before rendering), you drain the event queue with `xrPollEvent`. Each event you receive might require you to change your behavior — start rendering, stop rendering, recreate the session, or shut down.

```cpp
void PollEvents(bool& exitLoop, bool& requestRestart) {
    XrEventDataBuffer event{XR_TYPE_EVENT_DATA_BUFFER};
    while (xrPollEvent(instance, &event) == XR_SUCCESS) {
        switch (event.type) {
            case XR_TYPE_EVENT_DATA_SESSION_STATE_CHANGED:
                HandleSessionStateChange(event, exitLoop, requestRestart);
                break;
            case XR_TYPE_EVENT_DATA_INSTANCE_LOSS_PENDING:
                exitLoop = true;
                break;
            // ... other event types
            default:
                break;
        }
        event = {XR_TYPE_EVENT_DATA_BUFFER}; // reset for next poll
    }
}
```

Key detail: `XrEventDataBuffer` is a fixed-size union-like struct. You reset it to `{XR_TYPE_EVENT_DATA_BUFFER}` before each poll call. When `xrPollEvent` fills it in, reinterpret it as the specific event type using its `.type` field.

---

## XrEventDataSessionStateChanged

This is the event you'll handle most. It tells you that the session has moved to a new state:

```cpp
void HandleSessionStateChange(const XrEventDataBuffer& event,
                               bool& exitLoop, bool& requestRestart)
{
    const auto& stateEvent =
        reinterpret_cast<const XrEventDataSessionStateChanged&>(event);

    sessionState = stateEvent.state;  // store the new state

    switch (sessionState) {
        case XR_SESSION_STATE_READY:
            xrBeginSession(session, &beginInfo);
            isSessionRunning = true;
            break;

        case XR_SESSION_STATE_STOPPING:
            xrEndSession(session);
            isSessionRunning = false;
            break;

        case XR_SESSION_STATE_LOSS_PENDING:
            // Runtime is about to be lost (e.g. runtime update)
            // Destroy and optionally recreate everything
            requestRestart = true;
            exitLoop = true;
            break;

        case XR_SESSION_STATE_EXITING:
            // User or OS ended the session — clean up and exit
            exitLoop = true;
            break;

        default:
            break;
    }
}
```

---

## The Full State Diagram

```
  xrCreateSession()
        │
        ▼
     UNKNOWN
        │  (runtime initializing)
        ▼
      IDLE ◄────────────────────────────────────┐
        │  (runtime ready for you to begin)       │
        ▼                                         │
      READY                                       │
        │  xrBeginSession()                       │
        ▼                                         │
  SYNCHRONIZED ──────────────────────────────────┤
        │  (frames visible)                       │ (loss)
        ▼                                         │
    VISIBLE                                       │
        │  (input focus granted)                  │
        ▼                                         │
    FOCUSED  ◄─── normal operating state          │
        │                                         │
        │  (focus lost — goes back to VISIBLE)    │
        ▼                                         │
    VISIBLE                                       │
        │  (display hidden — goes back to SYNC)   │
        ▼                                         │
  SYNCHRONIZED                                    │
        │  runtime says stop                      │
        ▼                                         │
    STOPPING                                      │
        │  xrEndSession()                         │
        └────────────────────────────────────────►┘

        OR:

    LOSS_PENDING ──► (destroy & optionally recreate)
    EXITING      ──► (destroy & exit cleanly)
```

The runtime drives these transitions. You never set state directly — you only react.

---

## What to Do in Each State

| State | What it means | What you do |
|---|---|---|
| `IDLE` | Session created, runtime not ready | Nothing — wait for events |
| `READY` | Runtime wants you to start | Call `xrBeginSession` |
| `SYNCHRONIZED` | Submitting frames, may not be shown | Submit frames, no input |
| `VISIBLE` | Frames are visible | Submit frames, no input |
| `FOCUSED` | Full active state | Submit frames + process input |
| `STOPPING` | Runtime wants you to stop | Call `xrEndSession` |
| `LOSS_PENDING` | Runtime about to be lost | Destroy session, plan restart |
| `EXITING` | App should exit | Destroy everything, exit |

---

## Other Events to Handle

Beyond session state, poll for:

```cpp
case XR_TYPE_EVENT_DATA_INSTANCE_LOSS_PENDING: {
    // The XrInstance itself is being lost. Save state and exit.
    // There's no recovery — recreate from scratch.
    exitLoop = true;
    break;
}

case XR_TYPE_EVENT_DATA_INTERACTION_PROFILE_CHANGED: {
    // The user's active controller changed (e.g. swapped from Touch to hand tracking).
    // Re-query interaction profiles.
    break;
}

case XR_TYPE_EVENT_DATA_REFERENCE_SPACE_CHANGE_PENDING: {
    // Tracking origin is shifting (e.g. guardian reset, recentering).
    // Useful for knowing when to reposition your world anchor.
    break;
}
```

Unhandled event types should be silently ignored (the `default: break` case). New runtimes and extensions add new event types; ignoring unknowns is correct behavior.

---

## The Main Loop Structure

Putting it all together, your main loop looks like this:

```cpp
bool exitLoop = false;
bool requestRestart = false;

while (!exitLoop) {
    // 1. Handle Android/OS events (NativeActivity callbacks)
    ProcessAndroidEvents();

    // 2. Poll OpenXR events
    PollEvents(exitLoop, requestRestart);
    if (exitLoop) break;

    // 3. Render, but only when the session is in a running state
    if (isSessionRunning) {
        RenderFrame();
    }
}

// Cleanup...
```

The key discipline: **only call render functions when `isSessionRunning` is true.** Calling `xrWaitFrame` or `xrBeginFrame` outside of a running session will fail.

---

## The isSessionRunning Flag

You set this flag in response to state transitions:

```cpp
bool isSessionRunning = false;

// Set to true when: XR_SESSION_STATE_READY → call xrBeginSession → true
// Set to false when: XR_SESSION_STATE_STOPPING → call xrEndSession → false
```

`SYNCHRONIZED`, `VISIBLE`, and `FOCUSED` are all "running" states — you're submitting frames in all three. The practical difference between them matters for input (only process input in `FOCUSED`) and for knowing whether frames are actually visible to the user.

---

## Destruction Order (Complete for Chapter 02)

```
xrDestroySession(session)                    // 02.4
xrDestroyDebugUtilsMessengerEXT(messenger)   // 02.2
xrDestroyInstance(instance)                  // 02.1
// then: VkDevice, VkInstance
```

---

## Quick Reference

| Concept | Key type/call |
|---|---|
| Poll for events | `xrPollEvent(instance, &eventBuffer)` |
| Event buffer type | `XrEventDataBuffer` |
| State change event | `XrEventDataSessionStateChanged` |
| Session state enum | `XrSessionState` |
| Is session running? | `SYNCHRONIZED` \| `VISIBLE` \| `FOCUSED` |
| Start rendering | `xrBeginSession()` when state == `READY` |
| Stop rendering | `xrEndSession()` when state == `STOPPING` |

---

**Next:** [02-E — Instance + Session + Event Loop](../exercises/02-instance-session-eventloop.md)  
**Then:** [03.1 — View Configurations & Stereo Setup](../03-graphics/01-view-configurations.md)