# Render Relay Protocol (RRP)

**The open standard for server-rendered streaming applications on TVs and displays.**

---

## The Problem

Smart TVs ship with app stores. Every service that wants to appear on your TV must build and maintain a separate app for Samsung, LG, Roku, Apple TV, Fire TV, and Android TV — each with different SDKs, different review processes, different update cycles, and different quality bars.

The result is a graveyard of abandoned apps, inconsistent experiences, and software that can never be updated without going through a gatekeeper.

This is not how the web works. On the web, you ship a URL.

## The Idea

RRP does for TV apps what HTTP did for desktop software.

A renderer renders everything — UI, video, menus, the full experience — and streams it to the display as video. The TV (or any RRP viewer) is a dumb terminal: it plays the stream and forwards remote control input back to the renderer. That's it.

```
Renderer     ──▶  Video Stream  ──▶  TV
TV Remote    ──▶  Input Events  ──▶  Renderer
```

The renderer owns all logic. The viewer never needs to be updated. One `rrp://` URL is all you need.

## Why This Works Now

- **Streaming video is solved.** LL-HLS delivers 1–3 second latency natively on every modern TV platform. For non-gaming applications this is imperceptible.
- **WebSocket is universal.** Every platform that can run a video player can open a WebSocket.
- **Rendering hardware is cheap.** A $150 mini PC in a closet can render a beautiful UI and stream it to every TV in a house simultaneously.
- **The web proved the model.** Nobody ships a Samsung-specific executable anymore. They ship a URL. RRP extends that idea to the living room.

## What It Looks Like

**For users:**
- Open the RRP app on your TV
- Your renderer appears automatically (mDNS, zero configuration)
- Or enter an `rrp://` URL for a hosted service
- Navigate with your existing remote
- Done

**For developers:**
- Build a renderer, render whatever you want, stream it out as LL-HLS
- Accept WebSocket connections and listen for key events
- No SDK. No app store. No review process. No per-platform builds.
- Ship updates instantly — your users see them the next time they connect

**For TV manufacturers:**
- Implement one simple viewer: WebSocket + video player + mDNS
- Your TV becomes compatible with every RRP service ever built
- No app ecosystem to maintain. No third-party quality problems to own.
- Firmware stays stable. Services update themselves.

## The Vision

Today a golf simulator company must ship and maintain separate apps for Roku, Apple TV, Fire TV, Samsung, and LG — just to appear on a TV. With RRP they ship one renderer, publish one URL, and it works everywhere there is a compliant viewer.

Tomorrow, if Samsung ships RRP support in firmware, that URL works on every Samsung TV ever made, forever, with no app store involved.

This is not a new idea. It is the web. We are just pointing it at the TV.

## Protocol Overview

RRP defines:

- **Discovery** — renderers advertise via mDNS (`_rrp._tcp.local`), viewers connect automatically on the LAN. Remote/hosted renderers use `rrp://` URLs.
- **Handshake** — renderer and viewer negotiate video format and capabilities
- **Input events** — standard remote keys (up/down/left/right/ok/back/playpause) over WebSocket
- **Video stream** — LL-HLS strongly recommended, format negotiated at connect time
- **Authentication** — OAuth 2.0 Device Authorization Grant (RFC 8628). No password entry on the TV — ever.
- **Extensions** — namespaced extension events for domain-specific hardware (launch monitors, sensors, gamepads)

[Read the full spec →](SPEC.md)

## Extension Specs

RRP is designed to outlive any particular use case. The extension system allows domain-specific hardware to speak a standard language without modifying the core protocol.

**Available extensions:**
- [`golf.frp`](extensions/GOLF.md) — tunnels [Flight Relay Protocol](https://github.com/flightrelay/spec) events and commands for launch monitor data

Anyone can propose extensions via this repository. Vendor-specific extensions use reverse-domain namespacing (`birdielabs.golf.frp`).

## Reference Implementation

A reference renderer is planned — a simple directory-based video streaming application built in Rust. Browse a folder of video files navigated entirely by remote, full UI rendered by the renderer, streamed as LL-HLS.

Repository: `renderrelay/example-server` (coming soon)

## Viewers

| Platform | Repository | Status |
|----------|------------|--------|
| Roku | `renderrelay/roku-app` | Planned |
| Apple TV | `renderrelay/appletv-app` | Planned |
| Web (browser) | `renderrelay/web-client` | Planned |

## Renderer SDKs

| Language | Repository | Status |
|----------|------------|--------|
| Rust | `renderrelay/sdk-rust` | Planned |
| Python | `renderrelay/sdk-python` | Planned |
| Node.js | `renderrelay/sdk-node` | Planned |

## View-Only and Passive Displays

Because the video stream is standard LL-HLS, any device that can play HLS can display an RRP stream — no RRP viewer required.

This enables **multi-screen setups with a single control point**:

```
RRP Renderer
    → One full RRP viewer (Roku, Apple TV) — input + video
    → VLC on a second screen              — video only
    → Browser with hls.js on a third      — video only
    → Any smart TV with an HLS player     — video only
```

The HLS stream URL is a plain HTTP endpoint. On a local LAN renderer with `"auth": "none"`, it is openly accessible to any player on the network. Share the URL and it just works.

**Example use cases:**
- Golf sim in the garage with a control screen at the hitting bay and a spectator screen across the room
- Broadcast the sim to a TV in another room while someone else is playing
- Record or capture the stream directly with FFmpeg or OBS

No RRP involvement needed on passive screens. No app to install. No configuration.

## TLS and Secure Connections

Remote/hosted RRP renderers should use `rrps://` (maps to `wss://`). For local LAN renderers, plain `rrp://` is fine.

The simplest path to TLS is **nginx as a reverse proxy** — nginx terminates TLS and forwards plain WebSocket traffic to your renderer. Your renderer never needs to handle TLS directly.

```nginx
server {
    listen 443 ssl;
    server_name mygolfsim.com;

    location /rrp {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

Viewer connects to `rrps://mygolfsim.com` → nginx terminates TLS → forwards as plain WebSocket to your renderer on port 8080. This is a standard pattern and requires no RRP-specific configuration.

## Status

RRP is in early draft. The spec is open for feedback.

- [x] Protocol spec draft
- [x] Golf launch monitor extension spec draft
- [ ] Reference renderer (Rust)
- [ ] Roku viewer
- [ ] Apple TV viewer
- [ ] renderrelay.org

Feedback, issues, and pull requests welcome.

## License

The Render Relay Protocol specification is [CC0](LICENSE) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.

