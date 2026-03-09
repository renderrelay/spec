# RRP Golf Extension (`golf.frp`)
**Version:** 0.1.0-draft
**Repository:** github.com/renderrelay/spec
**Depends on:** [Flight Relay Protocol (FRP)](https://github.com/flightrelay/spec)

---

## Overview

**Namespace:** `golf.frp`

Tunnels [Flight Relay Protocol](https://github.com/flightrelay/spec) events and commands over an RRP WebSocket connection. FRP defines the message kinds, field names, unit conventions, and shot lifecycle. This extension defines only how FRP messages are carried inside the RRP `ext` envelope — it does not redefine or subset FRP.

As FRP evolves (new event kinds, new fields), those additions are automatically valid inside this extension with no RRP spec changes required.

---

## FRP Roles over RRP

FRP defines two roles independent of transport direction: the **device** (launch monitor or bridge that emits shot data and accepts commands) and the **controller** (simulator or application that receives shot data and sends commands). When tunneled through RRP, these roles are decoupled from RRP's renderer/viewer:

- **Relay** — the renderer acts as the FRP device, forwarding launch monitor data to the viewer (controller). Events flow renderer → viewer.
- **Direct hardware** — a launch monitor connects as a viewer and acts as the FRP device, sending events to the renderer (controller). Events flow viewer → renderer.

The message format is identical in both cases. Only the transport direction changes — the FRP roles do not.

---

## Envelope

FRP messages are carried in the standard RRP `ext` envelope. The FRP envelope fields (`device`, `event`) are merged into `data`:

```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "device": "EagleOne-X4K2",
    "kind": "ball_flight",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    },
    "ball": {
      "launch_speed": "67.2mps",
      "launch_azimuth": -1.3,
      "launch_elevation": 14.2,
      "carry_distance": "180.5m",
      "total_distance": "195.0m",
      "roll_distance": "14.5m",
      "max_height": "28.3m",
      "flight_time": 6.2,
      "backspin_rpm": 3200,
      "sidespin_rpm": -450
    }
  }
}
```

- `type` — always `"ext"`
- `ext` — always `"golf.frp"`
- `data` — the FRP payload. Contents depend on role (see below).

---

## Handshake

The FRP handshake (`start`/`init`) is performed inside the tunnel after the RRP handshake completes. The FRP controller sends `start` as an `ext` message, the FRP device responds with `init` as an `ext` message, then events and commands flow. The RRP handshake negotiates the RRP version and stream format; the FRP handshake negotiates the FRP version.

**FRP controller → FRP device:**
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "start",
    "version": ["0.1.0"],
    "name": "My Golf Sim"
  }
}
```

**FRP device → FRP controller:**
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "init",
    "version": "0.1.0"
  }
}
```

Note that handshake messages do not include `data.device` — the `device` field only appears on event envelopes after the handshake.

---

## Device Events

FRP device events (`shot_trigger`, `ball_flight`, `club_path`, `face_impact`, `shot_finished`, `device_telemetry`) carry launch monitor data. Sent by whichever RRP endpoint holds the FRP device role. The FRP envelope fields (`device`) and the FRP event fields (`kind` and all event-specific fields) are flattened into `data`:

| FRP envelope | RRP extension |
|-------------|---------------|
| `device` | `data.device` (required) |
| `event.*` | `data.*` (flattened into `data` alongside `device`) |

FRP messages without the `device`/`event` envelope (handshake messages, protocol-level alerts, and commands) place their top-level fields directly into `data`.

Either side may send FRP `alert` messages. Device-specific alerts include `data.device` (e.g. hardware warning on a particular launch monitor). Protocol-level alerts (e.g. incompatible FRP version, controller shutting down) omit `data.device` but still use the tunneled `ext` format. RRP's native `alert` message is reserved for RRP-level issues — FRP alerts must not be sent as RRP alerts.

**`device_telemetry` example** (no `key`, event fields flattened into `data`):
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "device": "EagleOne-X4K2",
    "kind": "device_telemetry",
    "manufacturer": "Birdie Labs",
    "model": "Eagle One",
    "firmware": "1.2.0",
    "telemetry": {
      "ready": "true",
      "battery_pct": "85"
    }
  }
}
```

**`alert` example — device-specific** (includes `data.device`):
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "device": "EagleOne-X4K2",
    "kind": "alert",
    "severity": "warn",
    "message": "Signal weak — reposition device"
  }
}
```

**`alert` example — protocol-level** (no `data.device`):
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "alert",
    "severity": "critical",
    "message": "Unsupported FRP version"
  }
}
```

---

## Controller Commands

FRP commands (`set_detection_mode`) control the launch monitor's detection parameters. Sent by the FRP controller to the FRP device. The FRP command payload is placed directly in `data`:

```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "set_detection_mode",
    "mode": "chipping",
    "handed": "rh"
  }
}
```

`data.device` is not present on commands.

---

## General

See the [FRP spec](https://github.com/flightrelay/spec) for the full message reference, field definitions, unit-tagged string format, and ShotAccumulator pattern.

The extension is a transparent passthrough — the RRP layer does not interpret or modify FRP fields. Both sides must ignore unknown `kind` values — this ensures forward compatibility as FRP evolves.

---

## License

The RRP Golf Extension specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved.

---

*RRP Golf Extension 0.1.0-draft — [github.com/renderrelay/spec](https://github.com/renderrelay/spec)*
