# Flight Relay Protocol (FRP)
**Version:** 0.2.0-draft
**Repository:** github.com/flightrelay/spec

---

## Overview

Flight Relay Protocol (FRP) defines a minimal open standard for **golf launch monitor event streaming** over WebSocket.

FRP is intentionally narrow in scope. It defines only:
- The shot event lifecycle (trigger, launch, flight result, club data, face impact, completion)
- Device telemetry (ready state, battery, tilt)
- Detection mode commands (full, putting, chipping) with optional player handedness and club in play
- Which values a device measured, which a flight model produced and which launch values it assumed, and whether a physical launch monitor measured a device's shots
- How endpoints that forward events keep each shot's identity and drop duplicates
- Unit conventions for all physical measurements
- Optional raw protocol frames for debugging, sent only when the controller asks for them

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

- **Device** — the FRP endpoint that emits shot data and accepts commands. A device may be a launch monitor, a bridge that aggregates multiple monitors, or any system that speaks the device side of FRP. The role is named "device" because it represents a launch monitor to the controller — even when a bridge is in the middle. This aligns with the `"device"` field on every FRP event envelope. A device declares with `source` whether a physical launch monitor measured its shots (see Measured Values and Flight Results).
- **Controller** — the FRP endpoint that receives shot data and sends commands. A controller may be a golf simulator, a stat tracker, a video overlay, an automation layer, or any system that drives device behavior and processes shot events.

Roles are **entity names, not directions**. Both sides send and receive messages — devices emit events and accept commands, controllers send commands and receive events. Roles are also independent of transport direction: either role may open the WebSocket connection or accept it, and all four combinations are valid FRP.

Most commonly the device is the WebSocket server and the controller connects to it — a launch monitor serving a simulator on the local network. The reverse is equally valid: a device may open the connection to a controller that is listening. This suits a bridge reporting into a central bus, a device behind NAT that cannot accept inbound connections, or a monitor configured to report to a fixed endpoint. The transport direction is also reversed when FRP is embedded in another protocol (e.g. RRP's `golf.frp` extension).

Whichever side opens the connection, the roles do not change: the controller sends `start`, the device answers `init`, the device emits event envelopes, and the controller sends commands.

**Recommended default URL:** `ws://localhost:5880/frp`. Implementations should use this port and path unless the host system specifies otherwise. The `/frp` path ensures FRP is uniquely addressable on any WebSocket server without conflicting with other services. The port and path belong to whichever side listens, which is the device in the common case and the controller when the device opens the connection.

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
  "version": ["0.2.0"],
  "name": "My Dashboard"
}
```

- `kind` — required, must be `"start"`
- `version` — required, array of FRP versions the controller supports (e.g. `["0.2.0"]`). The device selects the highest mutually supported version. If no version is compatible, the device sends a `critical` alert and closes the connection.
- `name` — optional, human-readable controller identifier for device-side logging. Defaults to `"anonymous"`.
- `raw_frames` — optional boolean. `true` asks the device to send `raw_frame` events (see Raw Frames). Defaults to `false`; a controller that does not want raw frames omits the field. A device treats any value other than `true` as `false`.

### Device → Controller: `init`

```json
{
  "kind": "init",
  "version": "0.2.0"
}
```

- `version` — the FRP version selected by the device from the controller's proposed list. A controller that receives a version it did not propose sends a `critical` alert and closes the connection.

After the `init` response, the device streams events.

The controller sends `start` as soon as the WebSocket connection is established, without waiting to be prompted — including when the device opened the connection. The device does not signal readiness; a completed WebSocket handshake is the signal.

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
launch         ┐
flight_result  │
club_path      ├── any order, any may be absent
face_impact    ┘
    ↓
shot_finished
```

`launch`, `flight_result`, `club_path`, and `face_impact` may arrive in any order. Any of them may not arrive at all (hardware-dependent). `shot_finished` is always the terminal event for a shot.

Endpoints that forward events drop a repeated event (see Forwarding), and a controller keeps the first event of each kind for a shot (see ShotAccumulator Pattern), so a device sends each event of the lifecycle once per shot, with all the data it has for that part of the shot. A controller discards a shot whose `shot_finished` does not arrive within its accumulator timeout (see ShotAccumulator Pattern).

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
    },
    "timestamp": 1794063845123
  }
}
```

- `shot_id` — required. UUID v4, globally unique across sessions. Used for machine correlation. The device that detected the strike generates it; endpoints that forward the shot pass it through unchanged (see Forwarding).
- `shot_number` — required. Monotonic counter from the device, for human display only. When or whether to reset is up to the device — FRP does not define session boundaries. Controllers echo back what the device sent for easy correlation with the device's local state.
- `timestamp` — optional. When the device detected the strike: an integer number of milliseconds since the Unix epoch (1970-01-01T00:00:00Z), in UTC. The example is 2026-11-07T15:04:05.123Z.

`timestamp` comes from the device's clock. Clocks on different devices drift and may be set wrong, so a controller that compares timestamps, e.g. to recognize one strike reported by two devices, allows a tolerance of its own choosing. `timestamp` is the device's declaration, like every other field: a controller does not rely on it for security. Endpoints that forward the shot pass it through unchanged. A controller ignores a `timestamp` that is not a non-negative integer and still processes the trigger.

---

### `launch`

Ball launch measurement data available. May arrive in any order relative to other shot events.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "launch",
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

**Launch fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `launch_speed` | unit-tagged string | Ball speed at launch |
| `launch_azimuth` | degrees (float) | Horizontal launch angle. Positive = right of target |
| `launch_elevation` | degrees (float) | Vertical launch angle above horizontal |
| `backspin_rpm` | integer (RPM) | Backspin component. Positive = backspin, negative = topspin |
| `sidespin_rpm` | integer (RPM) | Sidespin component. Positive = curves ball rightward (slice for RH, hook for LH) |
| `total_spin_rpm` | integer (RPM) | Total spin rate. Never negative |
| `spin_axis` | degrees (float) | Tilt of the spin axis from horizontal. Positive = tilted right, curves ball rightward (same sense as `sidespin_rpm`) |

`spin_axis` is omitted when `total_spin_rpm` is 0.

`launch` carries launch values only, and every value in it is measured (see Measured Values and Flight Results). Carry, total distance and the other results of the ball's flight travel in `flight_result`. In the example, the device measured the spin components and translated them into the total and axis.

**Spin.** `backspin_rpm` + `sidespin_rpm` and `total_spin_rpm` + `spin_axis` are two descriptions of the same spin:

```
backspin_rpm   = total_spin_rpm × cos(spin_axis)
sidespin_rpm   = total_spin_rpm × sin(spin_axis)
total_spin_rpm = √(backspin_rpm² + sidespin_rpm²)
spin_axis      = atan2(sidespin_rpm, backspin_rpm)
```

- A device sends the pair it measures, and should also send the other pair, translated from it as above. Many controllers read only the components.
- A translated pair is measured, like the pair it comes from.
- A device translates one pair only from both values of the other pair; a `total_spin_rpm` of 0 needs no axis. A device that measures only part of a pair sends only what it has: backspin without sidespin means `backspin_rpm` alone, and a total spin without an axis means `total_spin_rpm` alone, with no `spin_axis`.
- When both pairs are present, they must agree to within rounding.
- A controller that receives one pair may compute the other.

---

### `flight_result`

Results of the ball's flight: where it landed, where it came to rest, and how it flew. May arrive in any order relative to other shot events.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "flight_result",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    },
    "flight": {
      "basis": "modeled",
      "model": "Eagle One flight model 2.1 (Birdie Labs)",
      "carry_distance": "180.5m",
      "carry_offline": "-7.6m",
      "total_distance": "195.0m",
      "total_offline": "-8.4m",
      "roll_distance": "14.5m",
      "max_height": "28.3m",
      "flight_time": 6.2,
      "descent_angle": 38.4
    }
  }
}
```

**Flight fields:**

| Field | Type | Description |
|-------|------|-------------|
| `basis` | string | Required. `"tracked"` or `"modeled"`: how the values were obtained (see below) |
| `model` | string | Optional. Free-text name of the flight model and its producer |
| `carry_distance` | unit-tagged string | Carry distance (ball in air only) |
| `carry_offline` | unit-tagged string | Lateral offset of the landing point from the target line. Positive = right of target |
| `total_distance` | unit-tagged string | Total distance including roll |
| `total_offline` | unit-tagged string | Lateral offset of the rest position from the target line. Positive = right of target |
| `roll_distance` | unit-tagged string | Roll distance after landing |
| `max_height` | unit-tagged string | Maximum apex height |
| `flight_time` | float (seconds) | Time of flight (launch to landing) |
| `descent_angle` | degrees (float) | Angle of descent below horizontal at landing |
| `assumed` | object | Optional, `modeled` only. Launch values the flight model assumed because the shot's `launch` lacks them (see below) |

All fields except `basis` are optional; send what the device has. `carry_offline` and `descent_angle` describe the landing of a ball in flight; devices omit them for a ball that does not leave the ground (e.g. a putt).

**Basis.** Every `flight_result` says how its values were obtained:

| Value | Meaning |
|-------|---------|
| `tracked` | The device followed the ball's flight, and the values come from that tracking |
| `modeled` | A flight model produced the values, e.g. from the launch values |

A result that a flight model completed or extended, e.g. a tracked flight with a modeled landing and roll, is `modeled`. A launch monitor's own flight model counts as a flight model: carry that a camera unit computes internally from the launch it measured is `modeled`.

`model` names the flight model and who produced it, e.g. `"Eagle One flight model 2.1 (Birdie Labs)"`. It is free text for display and logging; controllers do not parse it.

A controller ignores a `flight_result` whose `basis` is missing or is not one of the values above.

**Assumed launch values.** A flight model needs a complete launch. When the shot's `launch` lacks a launch value the model needed, e.g. the spin of a short chip from hardware that measured its speed and angles but not its spin, the model assumes one. A `modeled` `flight_result` may carry those inputs in `assumed`, so that a controller that flies the ball itself can use the same values:

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "launch",
    "key": {
      "shot_id": "6f1c2a9e-3b7d-4e0a-9c58-1d2e3f4a5b6c",
      "shot_number": 43
    },
    "launch": {
      "launch_speed": "22.4mps",
      "launch_azimuth": 0.8,
      "launch_elevation": 31.5
    }
  }
}
```

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "flight_result",
    "key": {
      "shot_id": "6f1c2a9e-3b7d-4e0a-9c58-1d2e3f4a5b6c",
      "shot_number": 43
    },
    "flight": {
      "basis": "modeled",
      "model": "Eagle One flight model 2.1 (Birdie Labs)",
      "carry_distance": "31.2m",
      "carry_offline": "0.4m",
      "total_distance": "36.0m",
      "max_height": "5.1m",
      "assumed": {
        "backspin_rpm": 4500,
        "sidespin_rpm": 0
      }
    }
  }
}
```

- `assumed` holds any of the `launch` fields (`launch_speed`, `launch_azimuth`, `launch_elevation`, `backspin_rpm`, `sidespin_rpm`, `total_spin_rpm`, `spin_axis`), with the same units and signs as in `launch`. A sender omits `assumed` when it has nothing in it.
- An assumed value is not measured, so it never appears in `launch`. `assumed` holds only inputs the model assumed because the shot's `launch` lacks that field: a sender does not put a field in `assumed` that it sends in `launch` for the same shot.
- The spin rules of `launch` apply to the launch that `launch` and `assumed` describe together (see Spin): `assumed` may hold a partial pair, and when both pairs are complete they agree to within rounding.
- An assumed value is one the model used. An assumed 0, such as the sidespin in the example, is the model's input, not a placeholder.
- `assumed` belongs to a `modeled` result only. A controller ignores `assumed` in a `tracked` result and uses the rest of that result as usual.
- A controller that finds a field in both `launch` and `assumed` uses the `launch` value and ignores the assumed copy.
- A controller that flies the ball from the launch values uses each `launch` value, and fills a value `launch` lacks from `assumed` of a `modeled` result.

**One per shot.** A shot has at most one `flight_result`. It follows the lifecycle of the other shot events: it arrives in any order before `shot_finished`, and may be absent.

**Added by a forwarder.** An endpoint that forwards a shot (see Forwarding) may add a `flight_result` for it only when the device sent none. It sets `basis` to `modeled` and `model` to a name that identifies the forwarder's own model, and may include in `assumed` the launch values its model filled in. The device may send its `flight_result` at any point before `shot_finished`, so a forwarder adds one only after it has received the shot's `shot_finished`, and sends it before forwarding that `shot_finished`. The added event keeps the shot's `device` and `key` (see Forwarding).

Measured-only controllers ignore `flight_result`, and with it any `assumed` values (see Measured Values and Flight Results).

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
      "club_height": "0.12in",
      "spin_loft": 22.1,
      "low_point_distance": "1.6in"
    }
  }
}
```

**Club fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `club_speed` | unit-tagged string | Club head speed at impact |
| `club_speed_post` | unit-tagged string | Club head speed after impact |
| `path` | degrees (float) | Club path relative to target line. Positive = rightward (in-to-out for RH, out-to-in for LH) |
| `attack_angle` | degrees (float) | Attack angle. Positive = ascending (hit up) |
| `face_angle` | degrees (float) | Face angle at impact relative to the target line. Positive = rightward (open for RH, closed for LH) |
| `dynamic_loft` | degrees (float) | Effective loft at impact |
| `smash_factor` | float | Ball speed divided by club speed |
| `swing_plane_horizontal` | degrees (float) | Horizontal swing plane rotation. Positive = rightward (in-to-out for RH, out-to-in for LH) |
| `swing_plane_vertical` | degrees (float) | Vertical swing plane angle relative to ground |
| `club_offset` | unit-tagged string | Lateral position of club head at impact relative to ball center. Positive = toward toe side |
| `club_height` | unit-tagged string | Vertical position of club head at impact relative to ball center. Positive = above center |
| `spin_loft` | degrees (float) | Angle between the club head's direction of travel and the face normal at impact, in three dimensions |
| `low_point_distance` | unit-tagged string | Position of the lowest point of the club head's arc along the target line, relative to the ball. Positive = target side of the ball, negative = behind the ball |

Devices send `spin_loft` only as a measured value. `dynamic_loft − attack_angle` approximates it but ignores club path and face angle; that approximation is left to controllers (see Derived Values).

---

### `face_impact`

Club face impact location — where on the club face the ball was struck. May arrive in any order relative to `launch` and `club_path`. Not all hardware can measure face impact; controllers must handle its absence.

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
    "source": "measured",
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
- `source` — optional, `"measured"` | `"virtual"`. Where this device's shots originate (see Measured Values and Flight Results)
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

## Raw Frames

Raw frames are an optional debugging aid. A device that speaks another protocol to its launch monitor, or to its host, can show a controller the protocol messages behind its FRP events. FRP treats their content as opaque.

---

### `raw_frame`

One protocol message, as the device implementation framed it.

```json
{
  "device": "EagleOne-X4K2",
  "event": {
    "kind": "raw_frame",
    "dir": "rx",
    "seq": 1042,
    "timestamp": 1794063845123,
    "label": "result",
    "data": "0a1b2c",
    "key": {
      "shot_id": "550e8400-e29b-41d4-a716-446655440000",
      "shot_number": 42
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `dir` | string | Required. Direction, seen from the device implementation: `"rx"` for a message it received, e.g. from the launch monitor; `"tx"` for a message it sent, e.g. to the launch monitor |
| `seq` | integer | Required. Per-device counter: increases by one for every raw frame the device sends for that `device` |
| `timestamp` | integer | Optional. When the device implementation received or sent the message: milliseconds since the Unix epoch, in UTC, as for `shot_trigger` |
| `label` | string | Optional. Short text the device implementation chooses, e.g. the message's name |
| `data` | string | Required. The message's bytes as lowercase hexadecimal, two digits per byte |
| `key` | object | Optional. The shot the message belongs to, when the device knows it |

- **Off by default.** A device sends `raw_frame` events only to a controller that sent `"raw_frames": true` in its `start`, and sends none otherwise.
- **One message per frame.** `data` holds exactly one protocol message, framed as the device implementation frames it. FRP describes no format for it, and a controller does not rely on its content to process shots.
- **Identity.** A device does not send a raw frame that carries device or user identity, such as a serial number, a hardware address, a network name or account data. It applies its own exclusion before sending; a frame it is unsure of is not sent.
- **Order.** A device sends raw frames on the same connection as its other events, in the order it received or sent them relative to the events it builds from them: a frame comes before the shot event built from it.
- **Sequence.** `seq` lets a controller see gaps and lets a forwarder recognize a repeat (see Forwarding). A device keeps counting for as long as it uses the same `device` id, across connections where it can. A device whose counter restarts may have its first frames dropped by a forwarder that still remembers the earlier numbers.
- **Not shot events.** A raw frame is not part of the shot lifecycle, even when it carries a `key`. It does not create, complete or extend a shot, does not count toward first event wins or the accumulator timeout, and is not ignored or affected by a closed `shot_id`.
- A controller ignores a `raw_frame` whose `dir`, `seq` or `data` is missing or invalid, including a `data` that is not an even number of hexadecimal digits. It ignores a `timestamp` that is not a non-negative integer, a `label` that is not a string and a `key` that is not a valid shot key, and still uses the frame. A controller that does not use raw frames ignores them.

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

Tells the device what to expect for the next shots: detection mode, player handedness and club in play. The controller (simulator, automation layer) sends this to tell the device what type of shot is expected so it can tune its detection parameters accordingly. Radar launch monitors typically have distinct arming modes for full swings, chips, and putts, each optimized for different ball speeds and trajectories.

```json
{
  "kind": "set_detection_mode",
  "mode": "chipping",
  "handed": "rh",
  "club": "SW"
}
```

- `mode` — optional, `"full"` | `"putting"` | `"chipping"`. Shot detection mode.
- `handed` — optional, `"rh"` | `"lh"`. Player handedness. Some launch monitors use this to adjust tracking parameters (e.g. expected ball flight direction, camera positioning).
- `club` — optional, a club code from the table below. The club the player is about to hit.

All fields are optional and may arrive in separate messages. A controller might send `handed` once at session start, `mode` on every hole and `club` on every shot. Devices must latch each field independently — the most recent value for each field is the active value. Omitting a field does not reset it. `club` does not set or imply `mode`: a controller that wants putting detection sends `"mode": "putting"`, with or without `"club": "PT"`. If a device uses both and they conflict, `mode` governs detection.

`club` is a tracking hint, like `mode` and `handed`. Some launch monitors tune detection to the club in play: a unit may load a per-club profile before the shot, arm only when the club is a putter, or pass the club on to software that expects one. It describes the upcoming shot to the device; it is not a shot record, and devices do not echo it on shot events.

**Club codes:**

| Code | Club |
|------|------|
| `DR` | Driver |
| `MD` | Mini driver |
| `2W` – `9W` | Fairway wood, 2 to 9 |
| `1H` – `9H` | Hybrid, 1 to 9 |
| `1I` – `9I` | Iron, 1 to 9 |
| `PW` | Pitching wedge |
| `GW` | Gap wedge |
| `SW` | Sand wedge |
| `LW` | Lob wedge |
| `PT` | Putter |

Codes are case-sensitive. A controller whose club list is finer than this set sends the closest code; which code is closest is the controller's choice (e.g. a 54° wedge as `SW`, a 10-iron as `PW`).

A device that receives a value it does not recognize for any of these fields — e.g. a `mode` other than the three above, a club code outside this table, or a value that is not a string — ignores that field: it keeps its latched value and still applies the other fields of the message. `null` is treated as the field being absent.

Detection mode, handedness and club are informational and for optimization only. Devices that do not support a requested mode, handedness or club must silently ignore the field. A device that recognizes a club code but has no matching setting may apply the nearest club it supports, or ignore the field.

---

## Derived Values

Controllers commonly display values that follow exactly from FRP fields. FRP does not carry them as fields. This section is informative; it fixes the formulas and signs so controllers agree.

| Value | Formula | Sign |
|-------|---------|------|
| Face to path | `face_angle − path` | Positive = face points right of the club path |
| Curve | `carry_offline − carry_distance × tan(launch_azimuth)` | Positive = ball curved rightward in flight |
| Roll offline | `total_offline − carry_offline` | Positive = ball rolled rightward |
| Spin loft (approximate) | `dynamic_loft − attack_angle` | — |

Curve and roll offline use `flight_result` values, together with `launch` values where the formula needs them. Distances are converted to a common unit before they are combined.

---

## Measured Values and Flight Results

Values in `launch`, `club` and `impact` are measured. Values that need an aero or flight model travel only in `flight_result`, whose `basis` says how they were obtained. `source` on `device_telemetry` says where a device's shots originate. Together they let a controller tell a launch monitor's readings apart from modeled results, and measured shots apart from virtual ones.

### Measured values

A measured value is one the device measured, or a simple translation of one:
- converting between the two spin pairs (see Spin);
- converting a unit, precision or sign convention;
- applying a calibration offset, e.g. an impact height corrected for the club in use.

A value that needs no aero or flight model to produce is measured. A value that needs one is not, even when the launch monitor itself produces it: it is a flight result, and travels in `flight_result`. A launch value that a flight model assumed, e.g. a spin estimated from the launch speed and angle, travels only in `assumed` of a `modeled` `flight_result` (see Assumed launch values), never in `launch`. Any other modeled value that has no `flight_result` field is not sent.

### Placeholders are omitted

A device never sends a value it does not have. It omits the field instead of sending a placeholder, such as a `launch_elevation` of 0 from hardware that does not measure it, or a `spin_axis` of 0 when the axis is unknown. A controller can treat every value present as one the device has.

### `source`

`source` on `device_telemetry` declares where the shots of the device named in the envelope originate:

| Value | Meaning |
|-------|---------|
| `measured` | A physical launch monitor measured the shots |
| `virtual` | No physical measurement: simulators, mock devices, generated or test shots |

- `source` describes origin only. Relaying a shot, through any number of bridges, never changes it (see Bridges).
- `source` describes one device, not the connection. A bridge may carry a measured launch monitor and a virtual device side by side; it declares each one in that device's `device_telemetry`.
- A device's `source` does not change during a connection. A bridge whose input behind a `device` id changes source, e.g. a launch monitor replaced by a simulator, uses a new `device` id for it.
- A device that declares `source` includes it in every `device_telemetry` it sends for that device, and sends one before the device's first shot event.
- A controller uses the most recent `source` it has received for a device. Until it has received one, or when the value is not one of those above, the device's source is unknown. A controller that requires measured shots treats an unknown source as not measured.

### Bridges

- A bridge that forwards another FRP device passes that device's `source` through unchanged, and does not declare a `source` the upstream device did not declare. It may add simple translations of the measured values it forwards, e.g. the other spin pair, and nothing that needs a flight model. It adds a `flight_result` only as `flight_result` describes.
- A bridge that takes shots from an application that is not an FRP device declares their origin when it knows it. For example, a launch monitor's own companion application reports the launch monitor's measured shots, so a bridge reading them declares `measured`. When the bridge does not know the origin, it omits `source`. It sends the application's measured values in `launch`, `club` and `impact`, and the application's flight results in `flight_result`, with `basis` `modeled` unless it knows the launch monitor tracked them. It omits values the application fills in as placeholders.
- A bridge that reads a launch monitor's hardware itself declares `measured`. A bridge that generates shots declares `virtual`.
- A bridge that forwards another FRP device also follows Forwarding: it keeps the upstream `device`, `shot_id`, `shot_number` and `timestamp`, and drops duplicates.

### Measured-only controllers (informative)

A controller that needs measured data only, e.g. to record a tournament round:
- drops every shot from a device whose `source` is not `measured`, including an unknown source;
- ignores `flight_result` entirely, whatever its `basis`. It therefore never sees `assumed` launch values, which travel only there.

What the controller does with the remaining values is its own choice; for example, a simulator computes the ball's flight from the launch values with its own model. FRP defines no command for asking a device for measured-only data: how a controller enters or announces such a mode is outside this specification.

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

## Sign Conventions

Every side-to-side field uses one of two frames. Neither depends on player handedness, so a device never flips a sign because the player is left-handed.

| Frame | Positive | Fields |
|-------|----------|--------|
| **Target line** | Right of the target line, as seen from behind the ball looking at the target | `launch_azimuth`, `carry_offline`, `total_offline`, `sidespin_rpm`, `spin_axis`, `path`, `face_angle`, `swing_plane_horizontal` |
| **Club face** | Toward the toe | `face_impact.lateral`, `club_offset` |

Up-and-down fields are positive upward (`launch_elevation`, `attack_angle`, `face_impact.vertical`, `club_height`). `backspin_rpm` is positive for backspin. `low_point_distance` is positive on the target side of the ball.

Words such as open/closed, in-to-out/out-to-in, draw/fade and hook/slice depend on handedness. They are for display only: controllers derive them from the signed value and the player's handedness, and FRP never carries them. A device or bridge whose hardware or upstream protocol reports these values relative to the golfer converts them to the frames above before sending.

**Examples:**

| Shot | Right-handed | Left-handed |
|------|--------------|-------------|
| Strike 8 mm toward the toe | `lateral` = `"8mm"` | `lateral` = `"8mm"` |
| Face 2° open at impact | `face_angle` = `2.0` | `face_angle` = `-2.0` |
| Path 3° in-to-out | `path` = `3.0` | `path` = `-3.0` |
| Ball starts 1.5° left of target | `launch_azimuth` = `-1.5` | `launch_azimuth` = `-1.5` |
| Ball curves left (hook for RH, slice for LH) | `sidespin_rpm` < 0 | `sidespin_rpm` < 0 |

---

## ShotAccumulator Pattern

Controllers must not assume that `launch`, `flight_result`, `club_path`, and `face_impact` arrive in a fixed order, or that any of them arrive at all. The recommended pattern:

```
on shot_trigger:
    if shot_id is closed or already has an accumulator: ignore
    create accumulator keyed by shot_id

on launch:
    if accumulator[shot_id] already has launch data: ignore
    store launch data in accumulator[shot_id]

on flight_result:
    if basis is missing or unrecognized: ignore
    if accumulator[shot_id] already has flight data: ignore
    store flight data in accumulator[shot_id]

on club_path:
    if accumulator[shot_id] already has club data: ignore
    store club data in accumulator[shot_id]

on face_impact:
    if accumulator[shot_id] already has impact data: ignore
    store impact data in accumulator[shot_id]

on shot_finished:
    finalize accumulator[shot_id]
    emit composed ShotData (launch + flight + club + impact, any may be null)
    discard accumulator[shot_id]
    mark shot_id closed

on accumulator timeout (no shot_finished within T of shot_trigger):
    discard accumulator[shot_id] without emitting it
    mark shot_id closed

on any event for a closed shot_id:
    ignore
```

**First event wins.** A controller keeps the first `launch`, `flight_result`, `club_path` and `face_impact` of a shot and ignores a repeat, as a forwarder drops one (see Forwarding). A shot then carries the same data whichever route it took.

Storing the `flight` object whole keeps its `basis`, `model` and `assumed` with the values they describe. A controller that flies the ball itself reads the launch from the composed shot: each `launch` value, with a missing one filled from `flight.assumed` when `flight.basis` is `modeled`. A measured-only controller skips `flight_result` (see Measured Values and Flight Results).

**Accumulator timeout.** A controller discards an accumulator whose `shot_finished` has not arrived within its accumulator timeout `T` of the `shot_trigger`, and does not emit the discarded shot. The recommended default is 30 seconds. Some launch monitors deliver their full flight data several seconds after the strike, so a much shorter timeout discards shots that would have completed. A controller may choose another value.

**Closed shots.** A `shot_id` is closed once its shot has finished or been discarded. A controller ignores every event for a closed `shot_id`, `shot_finished` included, and does not start a new partial shot for it. This covers a late event for a discarded shot and the same shot arriving by a second route. A controller remembers a closed `shot_id` for at least `T` after it closed.

---

## Forwarding

An endpoint forwards events when it sends on events it received from another FRP endpoint: a bridge forwarding another FRP device, or a hub that passes events between connections. The rules here apply to every such endpoint, whichever role it plays on each connection.

**Shot identity.** A forwarder passes the envelope's `device` and the shot's `shot_id`, `shot_number` and `timestamp` through unchanged. It never generates a new `shot_id` for a shot it forwards. A controller can then recognize the same shot arriving by two routes.

**Duplicates.** A forwarder drops duplicates before forwarding, so a repeated event stops at the first endpoint that has already forwarded it, and an event that comes back around a loop dies out:
- **Shot events.** A forwarder drops a shot event whose `device`, `shot_id` and `kind` match one it already forwarded. It remembers forwarded shot events for a window at least as long as the accumulator timeout (see ShotAccumulator Pattern); 30 seconds matches the recommended default.
- **`device_telemetry`.** A forwarder forwards it only when it changes that device's last-forwarded state: a field or `telemetry` key whose value differs from the value last forwarded for that device. The first `device_telemetry` for a device always changes it. A forwarder still sends each device's current state in the `device_telemetry` it emits after a new handshake.
- **Alerts with a device context.** A forwarder drops an alert whose `device`, `severity` and `message` match one it forwarded within that window. A device that raises the same alert again after the window gets it forwarded again.
- **Raw frames.** A forwarder drops a `raw_frame` whose `device` and `seq` match one it forwarded within that window.
- A forwarder does not forward events of a `kind` it does not recognize.

**Raw frames.** A forwarder may drop raw frames instead of forwarding them. One that forwards them passes each unchanged, and sends them only to a controller that asked for them with `raw_frames` in its `start`; to receive them itself, it asks for them in its own `start`. Raw frames are not shot events: a forwarder does not track them per shot.

**Adding events.** A forwarder adds an event to a shot only to supply a missing `flight_result`, as `flight_result` describes. Because a forwarder drops a second `flight_result` for a shot like any other duplicate, a shot still carries at most one, even when it reaches a controller by several routes.

A transport that carries FRP messages without reading them, such as a tunnel, is not a forwarder; the endpoints at its ends are.

How forwarders are connected into a bus, how they discover each other and how events are routed between them are outside this specification.

---

## Robustness

- Unknown `kind` values must be silently ignored
- Unknown fields on known messages (events and commands) must be silently ignored
- Missing optional fields must be handled gracefully
- Invalid JSON must be silently ignored
- Controllers must not crash on partial shots (e.g. `launch` with no `flight_result`, `club_path` or `face_impact`)
- Controllers must handle shot events for unknown `shot_id` values gracefully (e.g. if the controller connected mid-shot and missed the `shot_trigger`)
- Both sides must handle a `critical` alert followed by connection close gracefully

---

## Versioning

The protocol version follows **semver**. Breaking changes increment the major version, or the minor version during the 0.x draft. Adding an optional field to an existing event or command is not a breaking change: peers ignore unknown fields and handle missing optional fields (see Robustness), and the version string stays the same.

0.2.0 is a breaking revision of 0.1.0: the `ball_flight` event is gone. Its launch values moved to the new `launch` event, and the results of the ball's flight to the new `flight_result` event. An endpoint implements exactly one major/minor version and proposes or accepts only that one, so a 0.2.0 endpoint and a 0.1.0 endpoint do not connect.

The controller proposes supported versions in `start` (`version` array). The device selects the highest mutually supported version and confirms it in `init` (`version` string). If no version is compatible, the device sends a `critical` alert and closes the connection.

Version strings in the handshake are plain semver (e.g. `"0.2.0"`). They carry no draft, release-candidate, or status suffix — the maturity of a given version is out-of-band knowledge between implementations.

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
- Accept `set_detection_mode` commands, latch each field independently, and silently ignore unsupported fields
- Ignore unknown `kind` values
- Send only measured values in `launch`, `club` and `impact`, and values that need a flight model only in `flight_result`, with its `basis`
- Omit every value it does not have, rather than sending a placeholder
- Send at most one `flight_result` per shot
- Send `assumed` only in a `modeled` `flight_result`, and only for launch fields the shot's `launch` does not carry
- When it is a bridge forwarding another FRP device, pass that device's `source` through unchanged
- Send `raw_frame` events only to a controller that asked for them with `raw_frames` in its `start`
- Never send a raw frame that carries device or user identity
- Increase `seq` by one for each raw frame it sends for a device, and send `data` as lowercase hexadecimal of exactly one message

A compliant FRP device should:
- Send a `critical` alert with a meaningful message before closing the connection, whether during handshake or mid-session
- Send each shot lifecycle event at most once per shot
- Send `timestamp` on `shot_trigger` when it has a clock set to UTC
- Send the other spin pair, translated from the pair it measures
- Name its flight model in `model` when it sends a `modeled` `flight_result`
- Declare `source` in `device_telemetry` for every device whose origin it knows
- Include `source` in every `device_telemetry` it sends for a device it declares, and send one before that device's first shot event

A **compliant FRP controller** (simulator, stat tracker, overlay, bridge) must:
- Complete the handshake
- Implement the ShotAccumulator pattern, including the accumulator timeout: discard a shot whose `shot_finished` does not arrive in time, and ignore events for a closed `shot_id`
- Handle missing `launch`, `flight_result`, `club_path`, or `face_impact` gracefully
- Handle `alert` messages gracefully, including connection close after `critical`
- Ignore unknown event kinds and unknown fields
- Keep only the first `launch`, `flight_result`, `club_path` and `face_impact` of a shot, and ignore a repeat
- Ignore a `flight_result` whose `basis` is missing or unrecognized
- Ignore `assumed` in a `tracked` `flight_result`, and use the `launch` value of a field that is in both `launch` and `assumed`
- Ignore a `timestamp` that is not a non-negative integer, and still process the trigger
- If it requires measured shots, treat a device whose `source` is absent or unrecognized as not measured, and ignore `flight_result`
- Keep `raw_frame` events out of the ShotAccumulator: they never create, complete or change a shot

A compliant FRP controller should:
- Send a `critical` alert with a meaningful message before closing the connection
- Use an accumulator timeout of 30 seconds unless it has a reason to choose another

A **compliant forwarder** (any device or controller that forwards events) must:
- Pass `device`, `shot_id`, `shot_number` and `timestamp` through unchanged, and never generate a new `shot_id` for a forwarded shot
- Drop duplicate shot events, `device_telemetry` that does not change the device's last-forwarded state, and repeated alerts with a device context, as described in Forwarding
- Add a `flight_result` to a forwarded shot only when the device sent none, with `basis` `modeled` and a `model` that names the forwarder's model; its `assumed` holds only launch values the forwarder's model filled in
- Pass a forwarded `raw_frame` unchanged, drop one whose `device` and `seq` match one it already forwarded within its window, and send raw frames only to a controller that asked for them

---

## License

The Flight Relay Protocol specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.

---

*FRP 0.2.0-draft — [github.com/flightrelay/spec](https://github.com/flightrelay/spec)*
