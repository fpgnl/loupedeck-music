# LoupedeckMusic — Product Requirements Document

> **Status:** Draft · **Version:** 0.2 · **Last updated:** 2026-03-22
> **Repository:** `github.com/fpgnl/loupedeck-music`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Problem Statement](#2-problem-statement)
3. [Goals & Non-Goals](#3-goals--non-goals)
4. [User Stories](#4-user-stories)
5. [System Architecture](#5-system-architecture)
6. [Technical Requirements](#6-technical-requirements)
   - 6.1 Loupedeck Bridge (daemon)
   - 6.2 Bridge Client (in-app)
   - 6.3 Apple Music Control
   - 6.4 Speaker Management
   - 6.5 Button Display
   - 6.6 Configuration System
7. [Tech Stack](#7-tech-stack)
8. [Out of Scope](#8-out-of-scope)
9. [Success Criteria](#9-success-criteria)
10. [Open Questions](#10-open-questions)

---

## 1. Overview

**LoupedeckMusic** is a native macOS application that bridges a
[Loupedeck Live](https://loupedeck.com/shop/loupedeck-live/) hardware console
with Apple Music, replacing the official Loupedeck software with a
purpose-built, low-latency driver and a modern visual configuration interface.

Each physical control (button, knob, touch strip) on the console is
user-configurable and displays a live, state-aware icon directly on the
hardware's built-in screens.

---

## 2. Problem Statement

| Pain point | Root cause |
|---|---|
| Commands lost or delayed | Official driver uses polling with no event buffering or sequencing |
| Sluggish UI response | Loupedeck software overhead |
| No Apple Music–specific integration | Generic plugin model |
| Complex, frustrating configuration UX | Loupedeck's proprietary config system |
| No multi-room speaker control | Not supported by official software |

The goal is to solve all five with a single focused application.

---

## 3. Goals & Non-Goals

### Goals

- **G1** — Sub-30 ms end-to-end latency from physical input to Apple Music action
- **G2** — Full playback control: play/pause, next, previous, seek, shuffle, repeat
- **G3** — Per-speaker volume control for every output listed in Apple Music's speaker menu (AirPlay 2, USB, built-in, analog)
- **G4** — Live icon feedback on Loupedeck touch keys reflecting current state (playing, volume level, active speaker)
- **G5** — Visual drag-and-drop configuration interface mirroring the console layout
- **G6** — Persistent, human-readable configuration (TOML/JSON) syncable across machines

### Non-Goals

- Cross-platform support (macOS only, initial release)
- Control of third-party music apps (Spotify, Tidal…) — future consideration
- Apple Music library browsing or playlist management
- Streaming / cloud API integration (Apple Music API)
- Replacing the Loupedeck firmware

---

## 4. User Stories

```
US-01  As a user, I can press a Loupedeck button to toggle an AirPlay speaker
       on or off in Apple Music with no perceptible delay.

US-02  As a user, I can turn a knob to adjust the volume of a specific speaker
       (or master volume) in real time.

US-03  As a user, I can see the current state of each speaker (active / muted /
       volume level) displayed as an icon on the corresponding touch key.

US-04  As a user, I can open a configuration window, see a graphical layout of
       my console, and drag an action + icon onto any button or knob.

US-05  As a user, I can choose from SF Symbols or upload a custom image for any
       button icon.

US-06  As a user, my configuration is stored in a plain-text file that I can
       version-control and sync to my other Macs.

US-07  As a user, I can mute/unmute Apple Music master output with a single
       button press.

US-08  As a user, I receive visual feedback on the console when a track changes
       (icon update, brief animation).
```

---

## 5. System Architecture

> **Interactive diagram:** [View on Excalidraw](https://excalidraw.com/#json=Nf2elEZPRw0_1pjPej1JU,kIvNun5kp8CxcrjZmgP0ZA)

```
┌──────────────────────────────────────────────────────┐
│  Loupedeck Live (hardware)                           │
│  WebSocket · USB RNDIS/ECM · 192.168.9.x:80          │
└──────────────────────┬───────────────────────────────┘
                       │ Binary WebSocket frames
                       ▼
┌──────────────────────────────────────────────────────┐
│  LoupedeckBridge  (background daemon / XPC service)  │
│                                                      │
│  • Owns the WebSocket connection exclusively         │
│  • Assigns sequence numbers to every inbound event   │
│  • Reconnects automatically on USB disconnect        │
│  • Heartbeat watchdog (ping every 500 ms)            │
│                                                      │
│  ZeroMQ PUSH tcp://127.0.0.1:5555  → events → app   │
│  ZeroMQ PULL tcp://127.0.0.1:5556  ← cmds  ← app   │
└──────────────────────┬───────────────────────────────┘
                       │ ZeroMQ PUSH/PULL (localhost)
                       │ — buffered, sequenced, no-drop
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  SwiftUI Application (Menu Bar + Configuration Window)          │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ConfiguratorView  — visual console layout, drag & drop   │  │
│  │  SpeakerGridView   — live speaker state display           │  │
│  └───────────────────────────┬───────────────────────────────┘  │
│                              │ @Published / Combine             │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │  AppCore (ObservableObject)                               │  │
│  │                                                           │  │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │  │
│  │  │  BridgeClient    │  │  MusicController             │  │  │
│  │  │  ZeroMQ PULL/    │  │  ┌──────────────────────┐    │  │  │
│  │  │  PUSH sockets    │  │  │ MediaRemote.framework│    │  │  │
│  │  │  AsyncStream     │  │  │ (transport commands) │    │  │  │
│  │  └────────┬─────────┘  │  ├──────────────────────┤    │  │  │
│  │           │            │  │ ScriptingBridge       │    │  │  │
│  │           │            │  │ (AirPlay speakers)   │    │  │  │
│  │           │            │  ├──────────────────────┤    │  │  │
│  │           │            │  │ CoreAudio            │    │  │  │
│  │           │            │  │ (USB/built-in devs)  │    │  │  │
│  │           │            │  └──────────────────────┘    │  │  │
│  │           │            └──────────────────────────────┘  │  │
│  │           │                                              │  │
│  │  ┌────────▼─────────────────────────────────────────┐   │  │
│  │  │  MappingEngine                                   │   │  │
│  │  │  button(id) / knob(id, delta) → Action           │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  │                                                           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  IconRenderer  — SF Symbol / custom → JPEG 90×90   │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Technical Requirements

### 6.1 Loupedeck Bridge (daemon)

| ID | Requirement |
|---|---|
| DR-01 | Run as a persistent background process (launchd `LaunchAgent`) independent of the UI app lifecycle |
| DR-02 | Communicate with hardware via WebSocket over the USB virtual network interface (RNDIS/ECM) at `192.168.9.x:80` |
| DR-03 | Parse binary event frames for: button down/up, knob rotate, touch strip swipe, wheel rotate |
| DR-04 | Assign a monotonically increasing sequence number to every inbound hardware event |
| DR-05 | Publish events on a **ZeroMQ PUSH** socket at `tcp://127.0.0.1:5555`; messages queue in-kernel if the app is slow or restarting — no events dropped |
| DR-06 | Accept display commands (JPEG payloads, LED colour) on a **ZeroMQ PULL** socket at `tcp://127.0.0.1:5556` |
| DR-07 | Send JPEG images (90×90 px) to individual touch keys with <16 ms round-trip from command receipt |
| DR-08 | Reconnect to hardware automatically on USB disconnect/reconnect with exponential back-off |
| DR-09 | Send WebSocket ping every 500 ms; treat two missed pongs as a disconnect |
| DR-10 | Reference implementation: [`foxxyz/loupedeck`](https://github.com/foxxyz/loupedeck) (Node.js, for protocol reference only) |

### 6.2 Bridge Client (in-app)

| ID | Requirement |
|---|---|
| BC-01 | Connect to ZeroMQ PULL socket (`5555`) on app launch; reconnect silently if the daemon restarts |
| BC-02 | Expose inbound events as a Swift `AsyncStream<LoupedeckEvent>` for consumption by `MappingEngine` |
| BC-03 | Detect out-of-order or missing sequence numbers and emit a warning log (recovery is stateless — no replay needed) |
| BC-04 | Send display commands via ZeroMQ PUSH socket (`5556`) from a dedicated actor to avoid back-pressure on the main queue |

### 6.3 Apple Music Control — Playback

| ID | Requirement | API |
|---|---|---|
| MC-01 | Play / Pause / Toggle | `MediaRemote.framework` — `MRMediaRemoteSendCommand` |
| MC-02 | Next / Previous track | `MediaRemote.framework` |
| MC-03 | Seek to position | `MediaRemote.framework` — `kMRSeekToPlaybackPosition` |
| MC-04 | Toggle Shuffle | `MediaRemote.framework` — `kMRToggleShuffle` |
| MC-05 | Toggle Repeat | `MediaRemote.framework` — `kMRToggleRepeat` |
| MC-06 | Now Playing push events | `MRMediaRemoteRegisterForNowPlayingNotifications` |
| MC-07 | Master volume / mute | `MediaRemote.framework` |

> Reference: [`nowplaying-cli`](https://github.com/kirtan-shah/nowplaying-cli) for MediaRemote usage patterns.

### 6.4 Speaker Management

| ID | Requirement | API |
|---|---|---|
| SP-01 | List all AirPlay speakers known to Music.app | `ScriptingBridge` → `Music.airPlayDevices()` |
| SP-02 | Activate / deactivate an AirPlay speaker | `ScriptingBridge` → `device.setActive()` |
| SP-03 | Set per-speaker volume (0–100) | `ScriptingBridge` → `device.setVolume()` |
| SP-04 | List CoreAudio output devices (USB, built-in, analog) | `AudioHardware` — `kAudioHardwarePropertyDevices` |
| SP-05 | Set CoreAudio device volume | `AudioObjectSetPropertyData` — `kAudioDevicePropertyVolumeScalar` |
| SP-06 | React to device plug/unplug events | `AudioObjectAddPropertyListenerBlock` |
| SP-07 | Poll AirPlay state every 2 s for UI sync (no push API) | `ScriptingBridge` on background queue |

> **Why ScriptingBridge over `osascript`:** in-process Apple Events, 5–30 ms latency vs 200–500 ms subprocess overhead.

### 6.5 Button Display

| ID | Requirement |
|---|---|
| BD-01 | Render SF Symbols to 90×90 px JPEG for any touch key |
| BD-02 | Support custom image import (PNG/JPEG, rescaled to 90×90) |
| BD-03 | Overlay dynamic state: volume bar, active indicator, short label (≤10 chars) |
| BD-04 | Re-render and push updated image to hardware on any state change |
| BD-05 | Icon background colour configurable per button |

### 6.6 Configuration System

| ID | Requirement |
|---|---|
| CF-01 | Config stored as TOML (human-readable, diff-friendly) |
| CF-02 | Each button/knob entry: `id`, `label`, `icon`, `action` |
| CF-03 | SwiftUI configurator: visual console layout, click-to-edit, drag icon from picker |
| CF-04 | Live preview: changes reflected on hardware in real time during config |
| CF-05 | Config path: `~/.config/loupedeck-music/config.toml` (XDG-style, easy to symlink/sync) |
| CF-06 | Each knob entry supports an optional `step` field (integer 1–20, default 5) defining the volume increment per detent |

---

## 7. Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Language | Swift 6 | First-class macOS API access, async/await, Sendable concurrency |
| UI | SwiftUI | Native macOS, modern declarative API |
| Hardware IPC | [ZeroMQ](https://zeromq.org) PUSH/PULL via [`SwiftyZeroMQ`](https://github.com/azawawi/SwiftyZeroMQ) or [`swift-zmq`](https://github.com/raycast/swift-zmq) | Kernel-buffered, no-drop local message bus; survives app restarts |
| Bridge daemon | Swift CLI / launchd `LaunchAgent` | Persistent background process, independent of UI lifecycle |
| Hardware WebSocket | `URLSession` WebSocket (daemon only) | Built-in, no dependencies; scoped to daemon |
| Apple Music playback | `MediaRemote.framework` (private) | System-level, push events, <5 ms |
| AirPlay speaker control | `ScriptingBridge` | In-process Apple Events, typed API |
| CoreAudio devices | `AudioToolbox` / `CoreAudio` | Direct hardware access, push notifications |
| Config format | TOML via [`TOMLKit`](https://github.com/LebJe/TOMLKit) | Human-readable, sync-friendly |
| Icon rendering | `CoreGraphics` + SF Symbols | No external assets needed |
| Distribution | Direct download / Homebrew cask | No App Store (private framework usage) |

---

## 8. Out of Scope

- Spotify, Tidal, or other music app support
- iOS / iPad companion app
- Apple Music library/playlist management
- Loupedeck firmware modification
- Windows or Linux support
- Multi-user / shared config scenarios

---

## 9. Success Criteria

| Metric | Target |
|---|---|
| Command latency (button → Music action) | < 30 ms p95 |
| Icon update latency (state change → hardware display) | < 100 ms |
| Zero lost input events under normal use (up to 20 events/s) | 100 % (ZeroMQ kernel buffer) |
| Config round-trip (edit → hardware preview) | < 500 ms |
| CPU usage at idle (no interactions) | < 1 % |
| macOS compatibility | Latest macOS as of 2026-03-22 and later (confirm version at project init) |

---

## 10. Open Questions

| # | Question | Answer |
|---|---|---|
| OQ-01 | Preferred repository name and GitHub org/account? | `github.com/fpgnl/loupedeck-music` |
| OQ-02 | Should the config UI live in the menu bar only, or also as a standalone window accessible from the Dock? | Both — menu bar icon for quick access + standalone configuration window openable from the Dock |
| OQ-03 | Should volume knob steps be fixed (e.g. ±5%) or user-configurable per knob? | User-configurable per knob (see CF-06) |
| OQ-04 | Target macOS minimum version? | Latest macOS as of 2026-03-22 — confirm exact version name at project init |
| OQ-05 | License preference (MIT, Apache 2.0, proprietary)? | MIT |

---

*Generated from design discussion on 2026-03-22.*
