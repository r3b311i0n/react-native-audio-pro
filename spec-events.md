# 🎧 React Native Audio Pro – Media Session & Playback State Contract

This document defines the contract between `react-native-audio-pro`'s native media session behavior, playback lifecycle states, and the TypeScript-facing API/events.

---

## 🧩 Method → State Flow

| Method | Description | State Transitions | Emits |
|--------|-------------|-------------------|--------|
| `configure()` | Set playback behavior (e.g., speech/music mode, debug) | _None_ | _None_ |
| `play(track, opts)` | Load & start track | `→ LOADING → PLAYING` or `→ LOADING → PAUSED` | `STATE_CHANGED: LOADING`, then `PLAYING` or `PAUSED` |
| `pause()` | Pause playback | `→ PAUSED` | `STATE_CHANGED: PAUSED` |
| `resume()` | Resume playback | `→ PLAYING` | `STATE_CHANGED: PLAYING` |
| `stop()` | Stop playback, reset position | `→ STOPPED` | `STATE_CHANGED: STOPPED` |
| `clear()` | Full teardown | `→ IDLE` | `STATE_CHANGED: IDLE` |
| `onError()` | Critical error | `→ ERROR` | `STATE_CHANGED: ERROR`, `PLAYBACK_ERROR` |
| `seekTo()` / `seekForward()` / `seekBack()` | Seek to new position (via TypeScript) | _No state change_ | `SEEK_COMPLETE` |
| `setPlaybackSpeed()` | Adjust speed | _No state change_ | `PLAYBACK_SPEED_CHANGED` |

> ⚠️ **All state transitions are emitted by native code only**. The TypeScript layer **must never infer or emit state**.
> Native is the single source of truth for the current playback state.

---

## 📊 Playback States

| State | Emitted When | Represents                                                                                |
|-------|--------------|-------------------------------------------------------------------------------------------|
| `IDLE` | After `clear()` | No track loaded, session removed                                                          |
| `STOPPED` | After `stop()` or `TRACK_ENDED` | Track loaded, session active, position reset to 0 (duration remains the track’s duration) |
| `LOADING` | After `play()` or mid-track buffer | Track fetching/buffering                                                                  |
| `PLAYING` | Playback is active | Audio playing, rate > 0                                                                   |
| `PAUSED` | After `pause()` or `play({ autoPlay: false })` | Audio paused, track retained                                                              |
| `ERROR` | After `onError()` | Failure encountered, all cleared                                                          |

> `IDLE` is never emitted on startup — it is assumed as the implicit default.
> `clear()` emits `STATE_CHANGED: IDLE`.
> `onError()` emits `STATE_CHANGED: ERROR`, and clears the player state just like `clear()`.
> `STOPPED` assumes position is reset to **0** and rate is **0**, but track and media session remain visible.

---

## 📡 Event Contract

| Event | Triggered By | Causes State Change? | Notes |
|-------|--------------|-----------------------|-------|
| `STATE_CHANGED` | Any state transition | ✅ | Emits `state`, `position`, `duration` |
| `PROGRESS` | Every second during playback | ❌ | Includes updated `position`, `duration` |
| `TRACK_ENDED` | When track completes | ✅ → `STOPPED` | Native must **pause**, **seek to 0**, then emit `STATE_CHANGED: STOPPED`. JS must not do this manually. |
| `SEEK_COMPLETE` | After a TypeScript-initiated seek completes | ❌ | Not emitted for native lock screen seeks |
| `PLAYBACK_SPEED_CHANGED` | On speed change | ❌ | Emits new rate |
| `REMOTE_NEXT` / `REMOTE_PREV` | Lock screen buttons | ❌ | Developer's app must handle and call `play()` |
| `PLAYBACK_ERROR` | On non-fatal error | ❌ | Separate from `STATE_CHANGED: ERROR`; may be emitted independently |

> ⚠️ **Playback errors and playback state are separate**.
> `PLAYBACK_ERROR` does **not imply** an `ERROR` state.
> Native must explicitly emit `STATE_CHANGED: ERROR` if the player enters an unrecoverable failure.

---

## 🎛 Buffering Behavior

If the player is **buffering mid-playback** (e.g. network stalls):
- Emit `STATE_CHANGED: LOADING`
- Do **not** emit `PAUSED` unless:
  - The user explicitly paused playback, or
  - Buffering fails entirely and triggers `onError()`

> Native must distinguish between **automatic buffering** and **user-intended pause**.

---

## 🔒 Lock Screen / Remote Control Behavior

| Control | Who Handles It | Emits Remote Event to JS? | Emits State Event? | Resulting State / Behavior |
|---------|----------------|----------------------------|--------------------|-----------------------------|
| ▶️ **Play** | Native (automatic) | ❌ | ✅ `STATE_CHANGED: PLAYING` | Resumes playback |
| ⏸ **Pause** | Native (automatic) | ❌ | ✅ `STATE_CHANGED: PAUSED` | Pauses playback |
| 📍 **Seek Bar** | Native (automatic) | ❌ | ✅ `STATE_CHANGED: PLAYING/PAUSED` in next `PROGRESS` | Seek updates position when lock-screen scrubbing is enabled; no `SEEK_COMPLETE` emitted |
| ⏭ **Next Track** | JS (developer's app) | ✅ `REMOTE_NEXT` | ❌ | App must call `play(nextTrack)` |
| ⏮ **Previous Track** | JS (developer's app) | ✅ `REMOTE_PREV` | ❌ | App must call `play(prevTrack)` |

> ⚠️ Native emits **all state changes** from lock screen interactions.
> TypeScript **does not emit state** — it only receives `STATE_CHANGED` or `REMOTE_*` where applicable.

> 🔧 When `allowLockScreenScrubbing` is set to `false` in `AudioProConfigureOptions`, native removes the seek bar / scrubbing command from lock screen and notification controls, so only play/pause and skip/next/prev remain available.

---

## 🔐 Summary: Ownership & Responsibility

- **Native (Swift/Kotlin)** is the **sole authority** on current playback state.
- **TypeScript receives** state via `STATE_CHANGED` — it does **not determine** state.
- `PLAYBACK_ERROR` is **not linked** to `STATE_CHANGED: ERROR`. They may occur together or separately.
- When a track ends, native is responsible for pausing, seeking to 0, and transitioning to `STOPPED`.
- Developers and agents working on this code must enforce strict alignment with this contract. Do not improvise or assume implicit behaviors — follow explicit transitions only.
