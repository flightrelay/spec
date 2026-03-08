# Flight Relay Protocol (FRP)
**Version:** 0.1.0-draft
**Repository:** github.com/flightrelay/spec

---

## Overview

Flight Relay Protocol (FRP) defines a minimal open standard for **golf launch monitor event streaming** over WebSocket.

FRP is intentionally narrow in scope. It defines only:
- The shot event lifecycle (trigger, ball flight, club data, face impact, completion)
- Device telemetry (ready state, battery, tilt)
- Detection mode commands (full, putting, chipping)
- Unit conventions for all physical measurements

FRP defines no transport discovery, no authentication, no configuration, no actor system, and no REST API. Those concerns belong to the systems built on top of it.

**FRP is the primitive.** Any system that needs to consume launch monitor data — a golf simulator, a stat tracker, a video overlay, a streaming bridge — can implement FRP without taking a dependency on any larger framework.

---

## Design Goals

- **Minimal** — implement in an afternoon in any language
- **Hardware-friendly** — a microcontroller bridging a radar sensor can speak FRP
- **Unit-safe** — all physical values are unit-tagged strings; no implicit unit assumptions
- **Composable** — designed to be embedded in larger protocols (see: [Flighthook](https://github.com/divotmaker/flighthook), [RRP golf.frp extension](https://github.com/renderrelay/spec/blob/main/extensions/GOLF.md))
- **Durable** — no dependencies on external standards that may change

---

## Transport

FRP messages are JSON text frames over WebSocket. The underlying WebSocket connection, port, path, TLS, and authentication are defined by the host protocol or application.

### Roles

FRP defines two roles:

- **Device** — the FRP endpoint that emits shot data and accepts commands. A device may be a launch monitor, a bridge that aggregates multiple monitors, or any system that speaks the device side of FRP. The role is named "device" because the data always originates from a physical device — even when a bridge is in the middle, it represents the device to the controller. This aligns with the `"device"` field on every FRP event envelope.
- **Controller** — the FRP endpoint that receives shot data and sends commands. A controller may be a golf simulator, a stat tracker, a video overlay, an automation layer, or any system that drives device behavior and processes shot events.

Roles are **entity names, not directions**. Both sides send and receive messages — devices emit events and accept commands, controllers send commands and receive events. Roles are also independent of transport direction. In standalone FRP, the device is typically the WebSocket server and the controller connects to it. When FRP is embedded in another protocol (e.g. RRP's `golf.frp` extension), the transport direction may be reversed — the device may be the WebSocket client — but the roles do not change.

**Recommended default URL:** `ws://localhost:5880/frp`. Implementations should use this port and path unless the host system specifies otherwise. The `/frp` path ensures FRP is uniquely addressable on any WebSocket server without conflicting with other services.

FRP does not define:
- How the WebSocket connection is established
- How the controller authenticates
- How the device is discovered

These are left to the host system. FRP defines only the message shapes that flow over the connection.

---

## Handshake

FRP requires a minimal handshake before event streaming begins.

### Controller → Device: `start`

```json
{
  "kind": "start",
  "version": ["0.1.0"],
  "name": "My Dashboard"
}
```

- `kind` — required, must be `"start"`
- `version` — required, array of FRP versions the controller supports (e.g. `["0.1.0"]`). The device selects the highest mutually supported version. If no version is compatible, the device sends a `critical` alert and closes the connection.
- `name` — optional, human-readable controller identifier for device-side logging. Defaults to `"anonymous"`.

### Device → Controller: `init`

```json
{
  "kind": "init",
  "version": "0.1.0"
}
```

- `version` — the FRP version selected by the device from the controller's proposed list

After the `init` response, the device streams events.

Messages sent before the `start` handshake are ignored.

If the device cannot accept the connection, it sends a `critical` alert (see Alert) and closes the WebSocket.

---

## Message Envelope

All device-to-controller messages after the handshake share this envelope (except `alert` messages without a device context, which omit the envelope — see Alert):

```json
{
  "device": "EagleOne-X4K2",
  "event": { "kind": "...", ... }
}
```

- `device` — identifier of the originating launch monitor. Every physical unit must have a unique `device` value. The recommended format is `"{model}-{serial}"` (e.g. `"EagleOne-X4K2"`). If the hardware does not expose a serial number or other stable unique identifier, the FRP device must generate a random ID (UUID recommended) and use it for the duration of the session.
- `event` — an `FrpEvent` tagged by `"kind"`

---

## Shot Lifecycle

Each shot produces a sequence of correlated events sharing a `shot_id`. Controllers must use the **ShotAccumulator** pattern: collect events until `shot_finished`, then process the composed shot.

```
shot_trigger
    ↓
ball_flight  ┐
club_path    ├── any order, any may be absent
face_impact  ┘
    ↓
shot_finished
```

`ball_flight`, `club_path`, and `face_impact` may arrive in any order. Any of them may not arrive at all (hardware-dependent). `shot_finished` is always the terminal event for a shot.

---

## Device Events

Device-to-controller shot event messages use the message envelope above and are tagged by the `"kind"` field on the `event` object.

---

### `shot_trigger`

Ball strike detected. Emitted immediately — no measurement data is available yet.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "shot_trigger",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    }
  }
}
```

- `shot_id` — required. UUID v4, globally unique across sessions. Used for machine correlation.
- `shot_number` — required. Monotonic counter from the device, for human display only. When or whether to reset is up to the device — FRP does not define session boundaries. Controllers echo back what the device sent for easy correlation with the device's local state.

---

### `ball_flight`

Ball flight measurement data available. May arrive in any order relative to other shot events.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
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

**Ball fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `launch_speed` | unit-tagged string | Ball speed at launch |
| `launch_azimuth` | degrees (float) | Horizontal launch angle. Positive = right of target |
| `launch_elevation` | degrees (float) | Vertical launch angle above horizontal |
| `carry_distance` | unit-tagged string | Carry distance (ball in air only) |
| `total_distance` | unit-tagged string | Total distance including roll |
| `roll_distance` | unit-tagged string | Roll distance after landing |
| `max_height` | unit-tagged string | Maximum apex height |
| `flight_time` | float (seconds) | Time of flight (launch to landing) |
| `backspin_rpm` | integer (RPM) | Backspin component. Positive = backspin, negative = topspin |
| `sidespin_rpm` | integer (RPM) | Sidespin component. Positive = right (slice for RH) |

---

### `club_path`

Club head measurement data available. May arrive in any order relative to other shot events.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "club_path",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    },
    "club": {
      "club_speed": "42.1mps",
      "club_speed_post": "38.6mps",
      "path": -2.1,
      "attack_angle": -3.5,
      "face_angle": 1.2,
      "dynamic_loft": 18.4,
      "smash_factor": 1.50,
      "swing_plane_horizontal": 5.3,
      "swing_plane_vertical": 58.1,
      "club_offset": "0.47in",
      "club_height": "0.12in"
    }
  }
}
```

**Club fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `club_speed` | unit-tagged string | Club head speed at impact |
| `club_speed_post` | unit-tagged string | Club head speed after impact |
| `path` | degrees (float) | Club path. Positive = in-to-out (draw for RH) |
| `attack_angle` | degrees (float) | Attack angle. Positive = ascending (hit up) |
| `face_angle` | degrees (float) | Face angle at impact. Positive = open (right of target) |
| `dynamic_loft` | degrees (float) | Effective loft at impact |
| `smash_factor` | float | Ball speed divided by club speed |
| `swing_plane_horizontal` | degrees (float) | Horizontal swing plane rotation. Positive = in-to-out (draw for RH) |
| `swing_plane_vertical` | degrees (float) | Vertical swing plane angle relative to ground |
| `club_offset` | unit-tagged string | Lateral position of club head at impact relative to ball center. Positive = toward toe side |
| `club_height` | unit-tagged string | Vertical position of club head at impact relative to ball center. Positive = above center |

---

### `face_impact`

Club face impact location — where on the club face the ball was struck. May arrive in any order relative to `ball_flight` and `club_path`. Not all hardware can measure face impact; controllers must handle its absence.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "face_impact",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    },
    "impact": {
      "lateral": "0.31in",
      "vertical": "0.15in"
    }
  }
}
```

**Impact fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `lateral` | unit-tagged string | Horizontal offset from face center. Positive = toward toe |
| `vertical` | unit-tagged string | Vertical offset from face center. Positive = above center |

---

### `shot_finished`

Shot sequence complete. Controllers should finalize and emit the accumulated shot.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "shot_finished",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    }
  }
}
```

---

## Device Status Events

These events are not part of the shot lifecycle and do not carry a `key` field.

---

### `device_telemetry`

Device identification and telemetry. Emitted immediately after the handshake completes, and re-emitted on changes.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "device_telemetry",
    "manufacturer": "Birdie Labs",
    "model": "Eagle One",
    "firmware": "1.2.0",
    "telemetry": {
      "ready": "true",
      "battery_pct": "85",
      "tilt": "0.5",
      "roll": "-0.2"
    }
  }
}
```

- `manufacturer`, `model`, `firmware` — all optional strings
- `telemetry` — optional key/value pairs, all string values

**Standard telemetry keys:**

| Key | Values | Description |
|-----|--------|-------------|
| `ready` | `"true"` / `"false"` | Device is ready to detect a shot (armed, ball detected, etc. — hardware-specific) |
| `battery_pct` | `"0"` – `"100"` | Battery percentage |
| `tilt` | degrees (string) | Device tilt angle |
| `roll` | degrees (string) | Device roll angle |

All telemetry values are strings. Devices may include additional keys beyond the standard set. Controllers should ignore unknown keys.

---

## Alert

Warning, error, or critical condition. Either side may send an `alert` at any time after the handshake, or during the handshake to explain a connection close.

```json
{
  "kind": "alert",
  "severity": "critical",
  "message": "Unsupported FRP version"
}
```

- `severity` — `"warn"` | `"error"` | `"critical"`
- `message` — human-readable description

How to interpret severity is up to the receiver. A `critical` alert indicates the sender cannot continue the session — the sender must close the connection after sending it.

Device-specific alerts (e.g. hardware warning on a particular launch monitor) use the standard message envelope — `device` and `event` wrapper:

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "alert",
    "severity": "warn",
    "message": "Signal weak — reposition device"
  }
}
```

Protocol-level alerts (from either side) omit the envelope entirely, placing `kind`, `severity`, and `message` at the top level as shown above.

The `alert` message is informational and optional — either side may close the connection at any time without sending one. Both sides must handle the connection closing with or without a preceding `alert` message.

---

## Controller Commands

Controller-to-device messages sent after the handshake. These do not use the message envelope.

---

### `set_detection_mode`

Sets the shot detection mode on the device. The controller (simulator, automation layer) sends this to tell the device what type of shot is expected so it can tune its detection parameters accordingly. Radar launch monitors typically have distinct arming modes for full swings, chips, and putts, each optimized for different ball speeds and trajectories.

```json
{
  "kind": "set_detection_mode",
  "mode": "chipping"
}
```

- `mode` — `"full"` | `"putting"` | `"chipping"`

Detection mode is informational and for optimization only — it hints to the device what type of shot is expected so it can tune detection parameters. Devices that do not support a requested mode must silently ignore the command.

---

## Unit-Tagged Strings

All velocity and distance fields are **unit-tagged strings**: a numeric value immediately followed by a unit suffix, with no space.

This makes the unit unambiguous regardless of what system the sender or receiver prefers, and eliminates implicit conversion bugs.

**Velocity:**

| Suffix | Unit |
|--------|------|
| `mps` | meters per second |
| `mph` | miles per hour |
| `kph` | kilometers per hour |
| `fps` | feet per second |

**Distance:**

| Suffix | Unit |
|--------|------|
| `m` | meters |
| `yd` | yards |
| `ft` | feet |
| `in` | inches |
| `mm` | millimeters |
| `cm` | centimeters |

**Examples:** `"67.2mps"`, `"150.3mph"`, `"180.5m"`, `"197.4yd"`, `"0.47in"`

Controllers must parse the suffix to interpret the value. Controllers must not assume a unit system. Devices should use the unit system native to their hardware and leave conversion to controllers or a conversion utility.

**Angles** are always decimal degrees (float). **Spin** is always RPM (integer). Neither is unit-tagged.

---

## ShotAccumulator Pattern

Controllers must not assume that `ball_flight`, `club_path`, and `face_impact` arrive in a fixed order, or that any of them arrive at all. The recommended pattern:

```
on shot_trigger:
    create accumulator keyed by shot_id

on ball_flight:
    store ball data in accumulator[shot_id]

on club_path:
    store club data in accumulator[shot_id]

on face_impact:
    store impact data in accumulator[shot_id]

on shot_finished:
    finalize accumulator[shot_id]
    emit composed ShotData (ball + club + impact, any may be null)
    discard accumulator[shot_id]

on timeout (no shot_finished within N seconds):
    discard stale accumulator
```

---

## Robustness

- Unknown `kind` values must be silently ignored
- Unknown fields on known events must be silently ignored
- Missing optional fields must be handled gracefully
- Invalid JSON must be silently ignored
- Controllers must not crash on partial shots (e.g. `ball_flight` with no `club_path` or `face_impact`)
- Controllers must handle shot events for unknown `shot_id` values gracefully (e.g. if the controller connected mid-shot and missed the `shot_trigger`)
- Both sides must handle a `critical` alert followed by connection close gracefully

---

## Versioning

The protocol version follows **semver**. Breaking changes increment the major version. The controller proposes supported versions in `start` (`version` array). The device selects the highest mutually supported version and confirms it in `init` (`version` string). If no version is compatible, the device sends a `critical` alert and closes the connection.

Version strings in the handshake are plain semver (e.g. `"0.1.0"`). They carry no draft, release-candidate, or status suffix — the maturity of a given version is out-of-band knowledge between implementations.

---

## Relationship to Other Standards

**[Render Relay Protocol (RRP)](https://github.com/renderrelay/spec)** `golf.frp` extension tunnels FRP events and commands over an RRP WebSocket connection, enabling any FRP-compatible launch monitor or bridge to relay shot data through an RRP renderer to any RRP viewer. The RRP extension carries FRP fields inside its `ext` envelope — FRP is the source of truth for all event and command definitions.

**[Flighthook](https://github.com/divotmaker/flighthook)** uses FRP as its core event model, extending it with a multi-device actor system, REST API, unit conversion, and configuration management.

---

## Compliance

A **compliant FRP device** (launch monitor or bridge) must:
- Complete the handshake
- Emit `shot_trigger` immediately on ball strike detection
- Emit `shot_finished` to terminate every shot sequence
- Use unit-tagged strings for all velocity and distance fields
- Emit `device_telemetry` immediately after the handshake, and re-emit on changes
- Accept `set_detection_mode` commands and silently ignore unsupported modes
- Ignore unknown `kind` values

A compliant FRP device should:
- Send a `critical` alert with a meaningful message before closing the connection, whether during handshake or mid-session

A **compliant FRP controller** (simulator, stat tracker, overlay, bridge) must:
- Complete the handshake
- Implement the ShotAccumulator pattern
- Handle missing `ball_flight`, `club_path`, or `face_impact` gracefully
- Handle `alert` messages gracefully, including connection close after `critical`
- Ignore unknown event kinds and unknown fields

A compliant FRP controller should:
- Send a `critical` alert with a meaningful message before closing the connection

---

## License

The Flight Relay Protocol specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.

---

*FRP 0.1.0-draft — [github.com/flightrelay/spec](https://github.com/flightrelay/spec)*
