# Render Relay Protocol (RRP)
**Version:** 0.1.0-draft
**Repository:** github.com/renderrelay/spec

---

## Overview

Render Relay Protocol (RRP) defines a minimal open standard for **server-side rendered streaming applications** controlled by remote input devices (TV remotes, gamepads, etc.).

The renderer owns all rendering logic. The viewer is a dumb terminal: it displays a video stream and forwards input events. Any compliant viewer works with any compliant renderer.

**Core use cases:**
- Living room / garage applications running on a powerful local machine
- Simulator software (golf, flight, racing) streamed to a TV
- Any application where rendering hardware and display are on separate devices

---

## Roles

RRP defines two roles:

- **Renderer** — the RRP endpoint that renders the application, serves the video stream, and processes input events. A renderer may be a gaming PC, a NAS, a cloud server, or any system that produces video and accepts remote input.
- **Viewer** — the RRP endpoint that displays the video stream and forwards input events. A viewer may be a Roku, Apple TV, Fire TV, browser, or any display with a WebSocket client and video player.

Roles are **entity names, not directions**. Both sides send and receive messages — renderers stream video and send notifications, viewers forward input and advertise capabilities. In standard RRP, the renderer is the WebSocket server and the viewer connects to it, but the roles are defined independently of transport direction to allow embedding in other protocols.

---

## Architecture

```
Physical Remote
    → Viewer (Roku, Apple TV, Fire TV, browser)
        → Input Events (WebSocket)
            → Renderer (gaming PC, NAS, cloud)
                → Renders frame
                    → Video Stream (LL-HLS or negotiated format)
                        → Viewer displays
```

---

## Discovery

### mDNS (Zero Configuration)
Renderers advertise themselves on the local network via mDNS:

- **Service type:** `_rrp._tcp.local`
- **TXT records:**
  - `name` — Human-readable renderer name (e.g. `"My Golf Sim"`)
  - `version` — RRP version (e.g. `"0.1.0"`)
  - `path` — WebSocket path (default `/rrp`)

Viewers scan for `_rrp._tcp.local` and present discovered renderers to the user. No configuration required.

### Manual URL (Fallback)
Any renderer can be addressed directly by URL:

```
rrp://192.168.1.50:8080
rrp://mygolfsim.com
```

The `rrp://` scheme maps to `ws://`. The `rrps://` scheme maps to `wss://` (TLS). This allows hosted renderers, remote renderers, and deep linking. Viewers must support manual URL entry as a fallback to mDNS discovery.

**Example:** A golf simulator company can publish `rrp://play.mygolfsim.com` and any viewer connects immediately.

---

## Connection Handshake

The handshake has four steps. Version negotiation happens first so that all subsequent messages are versioned.

### 1. Viewer → Renderer: `start`

The viewer initiates the connection with a version proposal:

```json
{
  "type": "start",
  "version": ["0.1.0"],
  "name": "roku"
}
```

- `type` — required, must be `"start"`
- `version` — required, array of RRP versions the viewer supports (e.g. `["0.1.0"]`). The renderer selects the highest mutually supported version. If no version is compatible, the renderer sends a `critical` alert and closes the connection.
- `name` — optional, human-readable viewer identifier for renderer-side logging. Defaults to `"anonymous"`.

Messages sent before the `start` handshake are ignored.

### 2. Renderer → Viewer: `init`

The renderer confirms the selected version and advertises its capabilities:

```json
{
  "type": "init",
  "version": "0.1.0",
  "name": "My Golf Sim",
  "auth": "none",
  "stream": {
    "supported": ["ll-hls", "hls", "dash", "webrtc", "rtsp"]
  },
  "input": {
    "keys": ["up", "down", "left", "right", "ok", "back", "playpause"]
  },
  "extensions": ["golf.frp"]
}
```

- `version` — the RRP version selected by the renderer from the viewer's proposed list
- `name` — optional, human-readable renderer name
- `auth` — `"none"` or `"required"` (see Authentication)
- `auth_endpoint` — required when `auth` is `"required"`. Base URL for the OAuth 2.0 Device Authorization Grant flow (see Authentication)
- `stream` — supported video formats
- `input` — accepted input keys
- `extensions` — optional, supported extension namespaces

Because `init` is sent after version negotiation, its schema can evolve with the protocol version.

### 3. Viewer → Renderer: `join`

The viewer selects a stream format and provides authentication if required:

```json
{
  "type": "join",
  "stream": {
    "selected": "ll-hls"
  },
  "extensions": ["golf.frp"]
}
```

- `stream.selected` — one of the formats from `init.stream.supported`
- `extensions` — optional, extensions the viewer supports
- `token` — optional, included only when the renderer requires authentication (see Authentication)

When the renderer requires authentication:

```json
{
  "type": "join",
  "stream": {
    "selected": "ll-hls"
  },
  "extensions": ["golf.frp"],
  "token": "rrp_tok_xxxxxxxx"
}
```

### 4. Renderer → Viewer: `stream_ready`

The renderer provides the stream URL for the selected format and confirms the active extension set:

```json
{
  "type": "stream_ready",
  "format": "ll-hls",
  "url": "http://192.168.1.50:8080/stream.m3u8",
  "extensions": ["golf.frp"]
}
```

- `format` — required, the confirmed stream format
- `url` — required, the stream URL for the selected format
- `extensions` — optional, the confirmed active extensions (intersection of renderer's `init.extensions` and viewer's `join.extensions`). Omitted or empty array means no extensions are active. Either side must ignore `ext` messages for extensions not in the confirmed set.

After `stream_ready`, the viewer begins playback and input events flow. The renderer may defer stream setup (e.g. starting an encoder) until the viewer's format selection is known.

---

## Input Events

Input events are sent as JSON over the WebSocket connection.

All events share a common envelope:

```json
{
  "type": "...",
  ...
}
```

- `type` — event type (see below)

Renderers must ignore unknown event types and unknown fields gracefully.

---

### Standard Remote Input

The core input type. All compliant viewers must support these.

```json
{
  "type": "key",
  "key": "left",
  "state": "down"
}
```

- `state` — `"down"` | `"up"` | `"repeat"` (held key, platform-dependent repeat rate)

#### Required Keys

Every compliant viewer must support these. Every compliant renderer must be fully controllable using only these keys — they are the common denominator across all remote input devices.

| Key | Description |
|-----|-------------|
| `up` | D-pad up |
| `down` | D-pad down |
| `left` | D-pad left |
| `right` | D-pad right |
| `ok` | Select / confirm |
| `back` | Back / escape |
| `playpause` | Combined play/pause toggle |

#### Optional Keys

Viewers should forward these when the remote supports them. Renderers may use them as shortcuts or optimizations but must not require them for core functionality.

| Key | Description |
|-----|-------------|
| `play` | Play (use when remote has separate play/pause buttons) |
| `pause` | Pause (use when remote has separate play/pause buttons) |
| `rewind` | Rewind |
| `fastforward` | Fast forward |
| `options` | Options / menu |
| `home` | Home (may not be forwardable on all platforms) |

---

### Extension Events

Extension events are **bidirectional** — both renderers and viewers may send them. This enables hardware viewers (launch monitors, sensors, gamepads) to push domain-specific data to the renderer, and renderers to send commands back (e.g. detection mode changes). Either side must ignore extension types it does not support.

Extensions are namespaced to avoid collisions:

```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": { ... }
}
```

- `type` — always `"ext"` for extension events
- `ext` — namespaced extension identifier (see Namespacing below)
- `data` — extension-specific payload

#### Extension Namespacing

Extensions use reverse-domain dot notation:

| Format | Example | Use |
|--------|---------|-----|
| `{domain}.{name}` | `golf.frp` | Open/community extensions |
| `{org}.{domain}.{name}` | `birdielabs.golf.frp` | Vendor-specific extensions |

The `golf.*` namespace is reserved for extensions defined in the RRP golf extension spec. Anyone may propose additions to this namespace via the renderrelay/spec repository.

#### Extension Advertisement

Both sides advertise supported extensions during the handshake. Renderers advertise in `init`:

```json
{
  "type": "init",
  "version": "0.1.0",
  "name": "My Golf Sim",
  "auth": "none",
  "stream": { ... },
  "input": { ... },
  "extensions": ["golf.frp"]
}
```

Viewers advertise in `join`:

```json
{
  "type": "join",
  "stream": { "selected": "ll-hls" },
  "extensions": ["golf.frp"]
}
```

Either side must ignore extensions it does not recognize.

#### Available Extensions

| Namespace | Spec | Description |
|-----------|------|-------------|
| `golf.frp` | [extensions/GOLF.md](extensions/GOLF.md) | Tunnels [Flight Relay Protocol](https://github.com/flightrelay/spec) events and commands for launch monitor data |

---

## Renderer Messages

Renderers may send messages to the viewer over the same WebSocket connection.

### Stream Ready
Sent after `join` to provide the stream URL for the negotiated format and confirm the active extension set. See Connection Handshake step 4.

### Stream Update
Sent if the stream URL changes mid-session. The format remains the one negotiated at handshake — only the URL may change.
```json
{
  "type": "stream_update",
  "url": "http://192.168.1.50:8080/stream2.m3u8"
}
```

### Notification (optional)
Display a brief message to the user:
```json
{
  "type": "notify",
  "message": "Club changed: 7 Iron",
  "duration": 2000
}
```

- `duration` — display time in milliseconds

Viewers may ignore notifications if display is not supported.

### Extension Events
Renderers may send `ext` messages to viewers using the same envelope as viewer extension events. See Extension Events above for the format.

### Ping / Keepalive
Standard WebSocket ping/pong frames are used for keepalive. No custom message needed.

---

## Alert

Warning, error, or critical condition. Either side may send an `alert` at any time after the handshake, or during the handshake to explain a connection close.

```json
{
  "type": "alert",
  "severity": "critical",
  "message": "Authentication required"
}
```

- `severity` — `"warn"` | `"error"` | `"critical"`
- `message` — human-readable description

How to interpret severity is up to the receiver. A `critical` alert indicates the sender cannot continue the session — the sender must close the connection after sending it.

The `alert` message is informational and optional — either side may close the connection at any time without sending one. Both sides must handle the connection closing with or without a preceding `alert` message.

---

## Authentication

RRP uses **OAuth 2.0 Device Authorization Grant (RFC 8628)** for authentication. This avoids password entry on remote-input devices entirely.

Renderers that require authentication must support this flow. Renderers that do not require authentication must advertise this so viewers skip the flow entirely.

### Init — Auth Field

The `init` message includes an `auth` field:

```json
{
  "type": "init",
  "version": "0.1.0",
  "name": "My Golf Sim",
  "auth": "none",
  ...
}
```

Or if authentication is required:

```json
{
  "type": "init",
  "version": "0.1.0",
  "name": "My Golf Sim",
  "auth": "required",
  "auth_endpoint": "https://mygolfsim.com/rrp/auth",
  ...
}
```

### Device Authorization Flow

When `auth` is `"required"` and no valid cached token is present:

**1. Viewer requests a device code**
```
POST {auth_endpoint}/device
```
Response:
```json
{
  "device_code": "abc123xyz",
  "user_code": "GOLF-4829",
  "verification_url": "https://mygolfsim.com/activate",
  "expires_in": 300,
  "interval": 5
}
```

**2. Viewer displays to user**

The viewer must display both the `user_code` and `verification_url` prominently on screen. The user visits the URL on any device (phone, computer) and enters the code. The viewer does not need to do anything else during this step.

**3. Viewer polls for token**

```
POST {auth_endpoint}/token
Body: { "device_code": "abc123xyz" }
```

Poll every `interval` seconds until one of:
- Token issued (success)
- `expires_in` elapsed (failure — restart flow)

Success response:
```json
{
  "access_token": "rrp_tok_xxxxxxxx",
  "token_type": "bearer",
  "expires_in": 2592000
}
```

**4. Viewer stores token locally**

The token is stored on the viewer, scoped to the renderer's hostname. Storage mechanism is platform-specific and opaque to the spec. The token must not be transmitted to any party other than the originating renderer.

**5. Token used on all subsequent connections**

The viewer includes the token in `join`:

```json
{
  "type": "join",
  "stream": { "selected": "ll-hls" },
  "token": "rrp_tok_xxxxxxxx"
}
```

If the token is valid, the session proceeds normally. If missing, expired, or invalid, the renderer sends a `critical` alert and closes the connection:

```json
{
  "type": "alert",
  "severity": "critical",
  "message": "Authentication required"
}
```

The viewer should restart the device authorization flow on reconnect.

### Token Expiry and Refresh

Renderers should issue long-lived tokens (30 days recommended for local LAN renderers) to minimize how often users must re-authenticate. Renderers may optionally include a `refresh_token` in the token response following standard OAuth 2.0 refresh semantics.

### Local LAN Renderers

Renderers on a local network (`.local` mDNS addresses) may set `"auth": "none"`. Authentication is most relevant for hosted/remote renderers accessible over the internet.

---

## Video Stream

### Strongly Recommended Default: LL-HLS
Low-Latency HLS (LL-HLS, RFC draft) is the strongly recommended default format:
- Natively supported by Roku, Apple TV, iOS, Android, and browsers
- 1–3 second latency — acceptable for non-gaming use cases
- No client-side decoder required

Renderers should support LL-HLS unless there is a specific reason not to.

### Negotiation
The `init` message allows renderers to advertise multiple formats and viewers to select based on capability in `join`. This ensures the protocol remains valid as video standards evolve.

**Format identifiers:**

| ID | Format |
|----|--------|
| `ll-hls` | Low-Latency HLS |
| `hls` | Standard HLS |
| `dash` | MPEG-DASH |
| `webrtc` | WebRTC (for sub-200ms latency) |
| `rtsp` | RTSP |

Future formats may be added without breaking the protocol.

---

## Ports and TLS

- **Port:** No default port. RRP runs over standard web ports (80, 443, 8080, etc.). The port is carried by mDNS SRV records and `rrp://` URLs.
- **Default path:** `/rrp`
- **TLS:** Supported via `rrps://` scheme (maps to `wss://`). Required for remote/hosted renderers.

---

## Robustness

- Unknown `type` values must be silently ignored
- Unknown fields on known messages must be silently ignored
- Invalid JSON must be silently ignored
- Viewers must handle `stream_update` gracefully (switch to new URL without interrupting the session)
- Both sides must handle the connection closing with or without a preceding `alert` message
- Renderers must ignore unknown input event types and unknown fields gracefully

---

## Compliance

A **compliant RRP viewer** must:
- Support mDNS discovery of `_rrp._tcp.local`
- Support manual `rrp://` URL entry
- Complete the handshake
- Forward all required keys (`up`, `down`, `left`, `right`, `ok`, `back`, `playpause`)
- Forward optional keys when the remote supports them
- Handle `alert` messages gracefully, including connection close after `critical`
- Display the negotiated video stream
- Ignore `ext` messages for extensions not in the confirmed set

A compliant RRP viewer should:
- Send a `critical` alert with a meaningful message before closing the connection

A **compliant RRP renderer** must:
- Be discoverable via mDNS and/or direct URL
- Complete the handshake
- Accept and process key events
- Be fully controllable using only the required keys
- Serve a valid video stream in at least one supported format
- Handle `alert` messages gracefully, including connection close after `critical`
- Ignore `ext` messages for extensions not in the confirmed set

A compliant RRP renderer should:
- Support LL-HLS as one of its offered formats
- Send a `critical` alert with a meaningful message before closing the connection

---

## Versioning

The protocol version follows **semver**. Breaking changes increment the major version. The viewer proposes supported versions in `start` (`version` array). The renderer selects the highest mutually supported version and confirms it in `init` (`version` string). If no version is compatible, the renderer sends a `critical` alert and closes the connection. All messages after `start` are versioned — their schemas may evolve across protocol versions.

Version strings in the handshake are plain semver (e.g. `"0.1.0"`). They carry no draft, release-candidate, or status suffix — the maturity of a given version is out-of-band knowledge between implementations.

---

## License

The Render Relay Protocol specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.

---

*RRP 0.1.0-draft — [github.com/renderrelay/spec](https://github.com/renderrelay/spec)*
