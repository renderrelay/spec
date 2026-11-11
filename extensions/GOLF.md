# RRP Golf Extension (`golf.frp`)
**Version:** 0.2.0-draft
**Repository:** github.com/renderrelay/spec
**Depends on:** [Flight Relay Protocol (FRP) 0.3.0](https://github.com/flightrelay/spec)

---

## Overview

**Namespace:** `golf.frp`

Tunnels the [Flight Relay Protocol](https://github.com/flightrelay/spec) bus over an RRP WebSocket connection. FRP defines the message kinds, field names, unit conventions, ownership rules and shot lifecycle. This extension defines only how FRP messages are carried inside the RRP `ext` envelope — it does not redefine or subset FRP.

As FRP evolves (new kinds, new fields), those additions are automatically valid inside this extension with no RRP spec changes required.

---

## One Connection, One FRP Bus Connection

An RRP connection that carries `golf.frp` is one FRP bus connection. Every FRP message sent or received on it travels as one RRP `ext` message.

FRP has no roles at the connection level: everything that takes part is an actor, and a connection can carry many actors. Either RRP endpoint may host actors, and RRP's renderer and viewer roles do not map to FRP roles. For example:

- **Relay** — the renderer hosts a game actor (and any other actors it runs). The viewer hosts a launch monitor actor, or relays the actors of a launch monitor it is attached to.
- **Direct hardware** — a launch monitor connects as an RRP viewer and hosts its own launch monitor actor. The renderer hosts the game and collectors.

The message format is identical in every case. Which endpoint holds which actor is not visible to the protocol.

---

## Envelope

An FRP message is a flat JSON object with `kind` and `from` at the top level. It is placed in `data` unchanged:

```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "shot.launch",
    "from": "7d2f9a41c03be856",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    },
    "launch": {
      "launch_speed": "67.2mps",
      "launch_azimuth": -1.3,
      "launch_elevation": 14.2,
      "backspin_rpm": 3200,
      "sidespin_rpm": -450,
      "total_spin_rpm": 3231,
      "spin_axis": -8.0
    }
  }
}
```

- `type` — always `"ext"`
- `ext` — always `"golf.frp"`
- `data` — one FRP message, exactly as FRP defines it. No field is added, removed, renamed or moved.

---

## Connecting

The FRP bus connection starts after the RRP handshake completes and `golf.frp` is among the confirmed extensions (`stream_ready.extensions`). FRP has no handshake of its own, and the RRP handshake carries no FRP version. Each actor states its FRP version in the `frp` field of its `bus.actor`.

Each endpoint runs this sequence for every actor it hosts:

1. Sends the actor's `bus.actor`.
2. Republishes the actor's state.
3. Sends `bus.sync`.

Every actor that receives `bus.sync` republishes all of its state, `bus.actor` first, so each endpoint rebuilds its view of the bus from what the other sends. An endpoint that hosts no actors sends nothing of its own.

After that, messages flow in both directions for as long as the connection lasts: state when it changes, occurrences such as `shot.launch` as they happen, commands to their owners, and each actor's `bus.actor` every `heartbeat` seconds.

If the RRP connection drops and reconnects, it is a new FRP bus connection and the connect sequence runs again, after the new RRP handshake.

**Launch monitor actor, first message after the RRP handshake:**
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "bus.actor",
    "from": "7d2f9a41c03be856",
    "name": "Eagle One X4K2",
    "frp": "0.3.0",
    "heartbeat": 2.0,
    "roles": ["launch_monitor"],
    "blockers": [],
    "device": {
      "manufacturer": "Birdie Labs",
      "model": "Eagle One",
      "firmware": "2.4.1",
      "source": "measured",
      "battery_pct": 82,
      "tilt": 0.4,
      "roll": -0.2
    }
  }
}
```

**Then, after its state, the sync request:**
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": { "kind": "bus.sync", "from": "7d2f9a41c03be856" }
}
```

---

## Messages

All 21 FRP kinds (`bus.*`, `shot.*`, `golfer.*`, `practice.*`, `game.*`) and extension kinds `x.<vendor>.<name>` travel the same way: one message, one `ext`, the FRP object unchanged in `data`. State, occurrences and commands all use the same envelope. Commands carry the FRP `ref`, and the owner's `bus.reply` comes back as another `ext` message.

The RRP layer does not interpret or modify FRP fields. Both sides ignore an FRP `kind` they do not know and any fields they do not know, as FRP requires.

**`golfer.turn`** — the session's state, which a launch monitor reads as tracking hints:
```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "golfer.turn",
    "from": "c19e04b7a2d5f318",
    "golfer": "backnine.local-3",
    "club": "7I",
    "handedness": "rh",
    "mode": "full"
  }
}
```

**`shot.launch`** — see Envelope above.

---

## Alerts

FRP's `bus.alert` is an FRP message and travels as one, in `data`:

```json
{
  "type": "ext",
  "ext": "golf.frp",
  "data": {
    "kind": "bus.alert",
    "from": "7d2f9a41c03be856",
    "severity": "warn",
    "message": "Signal weak — reposition device"
  }
}
```

RRP's own `alert` message stays reserved for RRP-level conditions (RRP version, authentication, stream, extension negotiation). An FRP alert is never sent as an RRP `alert`, and an RRP-level condition is never sent as `bus.alert`. A `critical` RRP `alert` closes the RRP connection and with it the FRP bus connection. An FRP `bus.alert` with `severity` `critical` describes its sending actor, and it does not close the RRP connection.

---

## General

See the [FRP spec](https://github.com/flightrelay/spec) for the full message reference, field definitions, ownership and roles, blockers, rehydration, unit-tagged string format and the ShotAccumulator pattern.

Messages from one sender arrive in the order it sent them, as the RRP WebSocket guarantees. Nothing relies on order across senders.

---

## License

The RRP Golf Extension specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved.

---

*RRP Golf Extension 0.2.0-draft — [github.com/renderrelay/spec](https://github.com/renderrelay/spec)*
