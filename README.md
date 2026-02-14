# VROOM

**Virtual Remoting Over OpenMux**

*It's like having a Zoom call with your coding agent.*

---

## What is VROOM?

VROOM is a WebRTC-native protocol and runtime for interactive sessions with AI agents. You see what the agent sees (browser, terminal, desktop), hear it speak, talk to it by voice or text, and take over its screen with your mouse and keyboard — all in real-time over a single WebRTC connection.

VROOM is an application protocol built on [OpenMux](https://github.com/visionik/socketpipe/tree/openmux), a transport-agnostic channel multiplexing standard.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    WebRTC PeerConnection                 │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │  Video   │  │  Audio   │  │    OpenMux Channels   │  │
│  │  Track   │  │  Track   │  │                       │  │
│  │ (H264)   │  │ (Opus)   │  │  omux/control         │  │
│  │          │  │          │  │  omux/pointer          │  │
│  │ Agent's  │  │ Agent ←→ │  │  omux/button           │  │
│  │ screen   │  │ Human    │  │                       │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**One connection. Video, audio, and full bidirectional control.**

## Interaction Modes

VROOM supports three modes, switchable at runtime:

| Mode | Description | Input |
|------|-------------|-------|
| **View** | Watch the agent work. No input forwarded. | — |
| **Interact** | Remote desktop. Mouse and keyboard control the agent's browser/screen. | Mouse, keyboard, touch |
| **Voice** | High-level commands. "Click the login button", "scroll down", "type my email". | Voice, chat |

```
┌─────────┐     mode: view       ┌─────────┐
│         │◄─────── video ───────│         │
│  Human  │     mode: interact   │  Agent  │
│         │──── mouse/keys ─────►│         │
│         │     mode: voice      │         │
│         │──── "click About" ──►│         │
└─────────┘                      └─────────┘
```

## OpenMux Channels

VROOM defines three channels on top of OpenMux:

### `omux/control` — Reliable, Ordered

Session control, mode switching, clipboard, navigation, high-level commands.

| Type | Name | Direction | Payload | Description |
|------|------|-----------|---------|-------------|
| `0x80` | HANDSHAKE | Both | JSON | Capabilities, viewport, version |
| `0x81` | MODE | C→S | JSON | `{mode: "view"\|"interact"\|"voice"}` |
| `0x82` | RESIZE | C→S | JSON | `{width, height, scale}` |
| `0x83` | CLIPBOARD | Both | JSON | `{text, direction: "set"\|"get"}` |
| `0x84` | NAVIGATE | C→S | JSON | `{url}` |
| `0x85` | COMMAND | C→S | JSON | `{action, selector, text, ...}` |
| `0x86` | CURSOR | S→C | JSON | `{type, x, y}` — remote cursor state |
| `0x87` | STATE | S→C | JSON | Agent state (thinking, browsing, idle) |

### `omux/pointer` — Unreliable, Unordered

High-frequency mouse movement. Loss-tolerant — a dropped move event is irrelevant once the next one arrives.

| Type | Name | Payload | Size |
|------|------|---------|------|
| `0x01` | MOUSE_MOVE | `x:u16, y:u16` | 4 bytes |
| `0x02` | MOUSE_MOVE_REL | `dx:i16, dy:i16` | 4 bytes |
| `0x03` | SCROLL | `dx:i16, dy:i16` | 4 bytes |

### `omux/button` — Reliable, Ordered

Discrete input events. Every click and keypress must arrive, in order.

| Type | Name | Payload | Size |
|------|------|---------|------|
| `0x10` | MOUSE_DOWN | `x:u16, y:u16, button:u8` | 5 bytes |
| `0x11` | MOUSE_UP | `x:u16, y:u16, button:u8` | 5 bytes |
| `0x12` | KEY_DOWN | `key_len:u8, key:utf8, code_len:u8, code:utf8, mods:u8` | ~12-20 bytes |
| `0x13` | KEY_UP | `key_len:u8, key:utf8, code_len:u8, code:utf8, mods:u8` | ~12-20 bytes |
| `0x14` | TOUCH_START | `id:u8, x:u16, y:u16, pressure:u8` | 6 bytes |
| `0x15` | TOUCH_MOVE | `id:u8, x:u16, y:u16, pressure:u8` | 6 bytes |
| `0x16` | TOUCH_END | `id:u8` | 1 byte |

### Keyboard Encoding

VROOM uses **DOM `key` and `code` values** — not X11 keysyms (n.eko/Selkies) or USB HID scancodes (JetKVM). This maps directly to browser APIs and Playwright's keyboard interface with zero translation.

```
KEY_DOWN: key="a", code="KeyA"       → page.keyboard.down("a")
KEY_DOWN: key="Enter", code="Enter"  → page.keyboard.down("Enter")
KEY_DOWN: key="Control", code="ControlLeft", mods=0x02
```

### Modifier Bitmask

```
Bit 0: Shift
Bit 1: Ctrl
Bit 2: Alt
Bit 3: Meta/Cmd
Bit 4: CapsLock
Bit 5-7: Reserved
```

### Mouse Button Codes

```
0: Left
1: Middle
2: Right
3: Back
4: Forward
```

## High-Level Commands (Voice Mode)

In Voice mode, the human issues natural-language or structured commands instead of raw mouse/keyboard input:

```json
{"action": "click", "selector": "text=Sign In"}
{"action": "type", "selector": "input[name=email]", "text": "me@example.com"}
{"action": "scroll", "direction": "down", "amount": 3}
{"action": "navigate", "url": "https://github.com"}
{"action": "back"}
{"action": "screenshot"}
```

These are sent on `omux/control` as COMMAND messages. The agent's runtime translates them to Playwright (or equivalent) API calls.

## VROOM Handshake

After OpenMux HELLO/WELCOME, VROOM exchanges its own handshake on `omux/control`:

**Client → Server:**
```json
{
  "type": "handshake",
  "version": "0.1.0",
  "capabilities": ["pointer", "keyboard", "touch", "clipboard", "command"],
  "viewport": {"width": 1024, "height": 1024, "scale": 1.0},
  "mode": "view"
}
```

**Server → Client:**
```json
{
  "type": "handshake",
  "version": "0.1.0",
  "capabilities": ["pointer", "keyboard", "clipboard", "command", "cursor"],
  "viewport": {"width": 1024, "height": 1024},
  "agent": {"name": "Vinston", "state": "idle"}
}
```

## Connection Flow

```
1. HTTP POST /offer — exchange SDP (video + audio + DataChannels)
2. ICE candidates exchanged
3. PeerConnection established:
   - Video track: agent screen → client (H264)
   - Audio track: bidirectional (Opus)
   - omux/control DataChannel opens
   - omux/pointer DataChannel opens
   - omux/button DataChannel opens
4. OpenMux HELLO/WELCOME on omux/control
5. VROOM HANDSHAKE on omux/control
6. Mode: view (default) — client sees video, hears audio
7. Client switches to interact/voice mode as needed
```

## Fallback

If WebRTC fails, VROOM falls back to WebSocket:

```
wss://host/vroom
```

All three OpenMux channels multiplex over the single WebSocket. Same frame format, same message types. Video degrades to server-pushed JPEG frames on omux/control (or a separate video channel). Audio uses a separate WebSocket or falls back to text-only.

## Reference Implementation

The reference implementation is [voxio-bot](https://github.com/visionik/voxio-bot) (being renamed to vroom-server), built on:

- **Pipecat** — AI pipeline framework (LLM, TTS, STT)
- **aiortc** — Python WebRTC implementation
- **Playwright** — Headless browser for agent screen rendering

## Design Lineage

VROOM's input protocol was designed after studying:

| Project | Transport | Encoding | Key Encoding | Insight Borrowed |
|---------|-----------|----------|-------------|-----------------|
| **n.eko** | WebSocket | JSON | X11 keysym | Named event types, debuggability |
| **Selkies-GStreamer** | DataChannel | CSV | X11 keysym | Compact hot-path, relative mouse |
| **JetKVM** | DataChannel | Binary | USB HID | Separate queues per input type |
| **SocketPipe** | WebSocket | Binary frames | N/A | Transport abstraction, framing |

## Status

VROOM is in active development. The protocol is draft and subject to change.

## License

MIT
