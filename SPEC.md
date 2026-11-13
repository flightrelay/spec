# Flight Relay Protocol (FRP)
**Version:** 0.3.0
**Repository:** github.com/flightrelay/spec

---

## Overview

Flight Relay Protocol (FRP) is an open bus for golf simulation data. Everything that takes part is an **actor**:
launch monitors, sessions, practice tools, games, flight models, lights and collectors (stat trackers, overlays,
UIs). Every actor publishes JSON messages to the bus and receives every message on it.

FRP defines:
- The messages: 21 kinds in five categories, `bus.*`, `shot.*`, `golfer.*`, `practice.*` and `game.*`
- Who owns each piece of state and who answers each command
- The shot lifecycle (trigger, launch, club data, face impact, flight, completion) and the ShotAccumulator
- Which values a launch monitor measured, which a flight model produced and which launch values it assumed
- Unit conventions and sign conventions for all physical measurements
- Optional raw protocol frames for debugging
- A WebSocket binding

FRP defines no discovery, no authentication and no configuration. Those concerns belong to the systems built on top
of it. Trust belongs to the transport (e.g. certificates over WSS).

### Design principles

- **Ownership.** Each piece of state has one owner. Only the owner publishes it, and only the owner answers commands
  that change it.
- **A message carries only what its sender owns.** A launch monitor sends what it measured; it never stamps the
  golfer or club on a shot, because it doesn't own them.
- **Joins happen at the consumer.** A collector combines a shot with the golfer up, the club and the practice block
  from their owners' messages. No actor republishes another actor's data.
- **No gap filling.** Transports are durable. A lost message is lost; nothing detects or replays it.
- **Rehydrate current state on reconnect.** An actor that connects or reconnects asks for the current state and
  carries on from there.
- **Unit-safe.** All speeds and distances are unit-tagged strings; no implicit unit assumptions.

---

## The Bus

### Envelope

Every message is one JSON object with its fields at the top level:

| Field | Type | Description |
|-------|------|-------------|
| `kind` | string | Required. The message kind, e.g. `shot.launch` |
| `from` | string | Required. The sending actor's id |
| ... | | The kind's own fields |

```json
{ "kind": "bus.sync", "from": "e4a7136b9f02cd58" }
```

- **Actor id.** A random string generated once by the host that runs the actor and kept across restarts. 16
  lowercase hexadecimal digits are recommended. It identifies the actor, never the machine or the person. `name` in
  `bus.actor` is for people.
- One connection can carry many actors; `from` is per message. A **hub** is a process that relays messages between
  connections (see WebSocket).
- A receiver ignores a message of a kind it doesn't know, and ignores fields it doesn't know. New optional fields and
  new kinds don't change the version.
- **Extensions** are kinds named `x.<vendor>.<name>`, with the same envelope.

### Message classes

Every kind belongs to one of three classes.

**State** is a complete object, never a delta. Its owner publishes it when it changes, when the owner starts, and in
answer to `bus.sync`. A state is identified by (`kind`, `from`), plus `id` for kinds that have several
(`golfer.profile`, `practice.block`). The newest message replaces the older one.

**Removing state.** The owner sends a last message for that state with `"removed": true` and nothing else but its
identity (`kind`, `from`, and `id` where the kind has one). Receivers forget it. When an actor goes away (see
`bus.actor`), receivers forget all its state. "Required" in the field tables applies to every message except a
removal, which carries only the state's identity and `removed: true`.

```json
{ "kind": "golfer.profile", "from": "c19e04b7a2d5f318", "id": "clubhouse.0f8c2d1a-77b4-4e1e-9a50-3c6d2b8e4f10", "removed": true }
```

**Occurrence**: sent once, never republished.

**Command**: names the thing it changes, not an actor. Each command kind has one owner role (see Roles and
Ownership). The owner applies it, publishes the changed state, then answers with `bus.reply`. A command with no
owner on the bus gets no answer; the sender stops waiting after a time of its own choosing. Every command carries
`ref`, a string the sender chooses (unique among its own open commands), echoed in the reply.

| Class | Kinds |
|-------|-------|
| State | `bus.actor`, `golfer.profile`, `golfer.turn`, `practice.block`, `game.round`, `game.scores`, `game.ball` |
| Occurrence | `bus.sync`, `bus.reply`, `bus.alert`, `shot.trigger`, `shot.launch`, `shot.club`, `shot.impact`, `shot.flight`, `shot.finished`, `shot.raw`, `game.stroke` |
| Command | `golfer.select`, `practice.new_block`, `game.aim` |

### Roles and Ownership

`bus.actor` lists the actor's roles. Roles that own things:

| Role | Owns | Answers |
|------|------|---------|
| `session` | `golfer.turn` | `golfer.select` |
| `practice` | `practice.block` | `practice.new_block` |
| `game` | `game.round`, `game.scores`, `game.ball`; sends `game.stroke` | `game.aim` |
| any actor listing a namespace | `golfer.profile` for ids in that namespace | |

Informative roles (no ownership): `launch_monitor`, `flight_model`, `lights`, `collector`. Other strings are allowed.

At most one actor on a bus holds each owning role, and at most one issues each namespace. Two claims are a
configuration error that UIs show to the user. The protocol has no priority rules. Until the user fixes it, which one
a receiver follows is its own choice.

One actor may hold several roles, e.g. `session` and `practice`.

### Heartbeats and Departure

Every actor republishes its `bus.actor` every `heartbeat` seconds, the interval it states in that message.

- A receiver that hears no `bus.actor` from an actor for three heartbeats treats it as gone and forgets all its
  state.
- An actor that stops cleanly sends its `bus.actor` with `"removed": true`.
- A returning actor sends `bus.actor` and republishes its state, as at start.
- What an actor does when an owner it relies on goes away is its own behavior. For example, a launch monitor carries on with
  the hints of the last `golfer.turn` it received; a UI shows the actor as gone.

### Rehydration

An actor that connects or reconnects sends `bus.sync`. Every actor that receives it republishes all of its state,
`bus.actor` first. The receiver rebuilds its view of the bus from those messages and carries on from there. Nothing
is replayed: occurrences sent while an actor was away are not sent again.

### Blockers

A blocker is a reason, given by the actor that has it, why a shot shouldn't be taken right now. Every actor lists
only its own, in `blockers` on its own `bus.actor`. `blockers` is the only readiness field in the protocol.

- **Display only.** No actor is required to act on another actor's blockers. A launch monitor publishes a shot hit
  while anything is blocked; each receiver decides what to do with it (e.g. a game ignores a shot when it isn't that
  golfer's turn).
- **"Ready to hit" is a join:** every actor's list is empty. UIs and lights compute it.
- An owner that refuses a command because of its own blockers answers `not_ready` with them (see `bus.reply`).
- Values are open strings. A receiver shows an unknown value as is. Standard values:

| Value | Typical actor | Meaning |
|-------|---------------|---------|
| `not_armed` | launch monitor | Not ready to detect a shot |
| `no_ball` | launch monitor | No ball seen in the hitting area |
| `paused` | session | The user paused play |
| `not_in_play` | game | No round or ball in play |
| `interrupted` | game | A menu, loading screen or similar is in the way |
| `not_your_turn` | game | The golfer up isn't playing at this simulator |
| `shot_in_progress` | game | The previous ball is still moving |
| `awaiting_decision` | game | The game is waiting on a ruling or a choice; see `game.ball` `decision` |
| `controls_disabled` | game | The game has turned off outside control |

### What Each Role Publishes and Reads (informative)

| Role | Publishes | Reads |
|------|-----------|-------|
| Launch monitor | `bus.actor` (with `device`), `shot.*` | `golfer.turn` |
| Flight model | `bus.actor`, `shot.flight` | `shot.launch` of the shot in flight |
| Session | `bus.actor`, `golfer.turn`, `golfer.profile` (its namespace), replies | `golfer.select` |
| Practice | `bus.actor`, `practice.block`, replies | `golfer.turn`, `practice.new_block` |
| Game | `bus.actor`, `game.*`, `golfer.profile` (its namespace), `golfer.select`, replies | `shot.*`, `game.aim` |
| Lights | `bus.actor` | occurrences; optionally `game.round` |
| Collector / UI | `bus.actor`, commands | everything; the only place joins happen |

---

## Bindings

The spec defines messages; a binding carries them over a durable, ordered transport. Messages from one sender arrive
in the order it sent them; nothing relies on order across senders.

### WebSocket

The one binding in 0.3.0.

- One message per WebSocket text frame, JSON.
- **Recommended default URL:** `ws://localhost:5880/frp`. Implementations should use this port and path unless the
  host system specifies otherwise. The `/frp` path keeps FRP uniquely addressable on any WebSocket server.
- **No handshake.** On connecting, each actor sends its `bus.actor`, republishes its state, then sends `bus.sync`. A
  hub that hosts no actors sends nothing of its own; it only forwards.
- **Hubs.** A hub forwards each message it relays unchanged to every other connection, never back to the one it came
  from. Which kinds it relays over a connection is its own configuration (default: all).
- Hubs connected in a loop would forward messages forever. A loop is a configuration error; FRP has no duplicate
  detection.

FRP does not define how the WebSocket connection is established, which side listens, how a peer authenticates, or
how peers discover each other. These are left to the host system.

---

## Values

### Unit-Tagged Strings

All velocity and distance fields are **unit-tagged strings**: a numeric value immediately followed by a unit suffix,
with no space.

This makes the unit unambiguous regardless of what system the sender or receiver prefers, and eliminates implicit
conversion bugs.

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

Receivers must parse the suffix to interpret the value. Receivers must not assume a unit system. Senders should use
the unit system native to their hardware and leave conversion to receivers or a conversion utility.

**Angles** are always decimal degrees (float). **Spin** is always RPM (integer). Neither is unit-tagged.
**Durations** are seconds (float). **Timestamps** are integer milliseconds since the Unix epoch
(1970-01-01T00:00:00Z), in UTC.

A placeholder is omitted, never sent as 0 (see Measured Values and Flight Results).

### Common Types

| Type | Definition |
|------|------------|
| **golfer id** | `<namespace>.<id>`, e.g. `backnine.local-3`. The namespace is lowercase letters, digits and `-`; the id after the first `.` is any non-empty string. Nobody publishes or translates ids in another actor's namespace |
| **distance** | A unit-tagged distance string, e.g. `"180.5m"` (see Unit-Tagged Strings) |
| **timestamp** | Integer milliseconds since the Unix epoch (1970-01-01T00:00:00Z), in UTC |
| **position** | An array of three unit-tagged distances `[x, y, z]` in the game's own world frame, z up. Only the game that sent it interprets x and y; others may compare positions from the same game |
| **handedness** | `"rh"` or `"lh"` |
| **mode** | `"full"`, `"chipping"` or `"putting"` |
| **club code** | A code from the table below. `null` where a club field allows it means no club (unknown or not applicable) |

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

Codes are case-sensitive. A sender whose club list is finer than this set sends the closest code; which code is
closest is the sender's choice (e.g. a 54° wedge as `SW`, a 10-iron as `PW`). A receiver treats an unknown code as
absent.

### Sign Conventions

Every side-to-side field uses one of two frames. Neither depends on player handedness, so a sender never flips a sign
because the player is left-handed.

| Frame | Positive | Fields |
|-------|----------|--------|
| **Target line** | Right of the target line, as seen from behind the ball looking at the target | `launch_azimuth`, `carry_offline`, `total_offline`, `sidespin_rpm`, `spin_axis`, `path`, `face_angle`, `swing_plane_horizontal`, `game.aim` `degrees`, `lie.slope_left_right` (positive = ground rises to the right) |
| **Club face** | Toward the toe | `impact.lateral`, `club_offset` |

Up-and-down fields are positive upward (`launch_elevation`, `attack_angle`, `impact.vertical`, `club_height`).
`backspin_rpm` is positive for backspin. `low_point_distance` is positive on the target side of the ball.
`lie.slope_up_down` is positive uphill toward the target.

Words such as open/closed, in-to-out/out-to-in, draw/fade and hook/slice depend on handedness. They are for display
only: receivers derive them from the signed value and the player's handedness, and FRP never carries them. A launch
monitor whose hardware or upstream protocol reports these values relative to the golfer converts them to the frames
above before sending.

**Examples:**

| Shot | Right-handed | Left-handed |
|------|--------------|-------------|
| Strike 8 mm toward the toe | `lateral` = `"8mm"` | `lateral` = `"8mm"` |
| Face 2° open at impact | `face_angle` = `2.0` | `face_angle` = `-2.0` |
| Path 3° in-to-out | `path` = `3.0` | `path` = `-3.0` |
| Ball starts 1.5° left of target | `launch_azimuth` = `-1.5` | `launch_azimuth` = `-1.5` |
| Ball curves left (hook for RH, slice for LH) | `sidespin_rpm` < 0 | `sidespin_rpm` < 0 |

---

## Fictional Actors in the Examples

| `from` | `name` | Roles |
|--------|--------|-------|
| `7d2f9a41c03be856` | Eagle One X4K2 | `launch_monitor` |
| `c19e04b7a2d5f318` | Clubhouse | `session`, `practice` |
| `5b80e2f7d14c9a63` | Back Nine Golf | `game` |
| `e4a7136b9f02cd58` | Fairway Physics | `flight_model` |

---

## Bus Messages (`bus.*`)

### `bus.actor`

**State, every actor.** Who the actor is, what it owns, and its status. Doubles as the heartbeat. Published at
start, on any change (e.g. `blockers`), in answer to `bus.sync`, and every `heartbeat` seconds even when nothing
changed.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Required. Display name |
| `frp` | string | Required. Protocol version, `"0.3.0"` (see Versioning) |
| `heartbeat` | float (seconds) | Required. The interval at which this actor republishes `bus.actor` |
| `roles` | array of string | Optional. See Roles and Ownership |
| `namespaces` | array of string | Optional. Golfer id namespaces this actor issues |
| `blockers` | array of string | Required, empty when the actor sees no reason a shot shouldn't be taken now. See Blockers |
| `device` | object | Optional. For launch monitors, below |
| `telemetry` | object | Optional. Free key/value pairs, string values, for display only |
| `removed` | `true` | Sent once when the actor stops cleanly |

**`device`:**

| Field | Type | Description |
|-------|------|-------------|
| `manufacturer`, `model`, `firmware` | string | Optional |
| `source` | string | Optional. `"measured"` or `"virtual"`: where this actor's shots originate (see `source`) |
| `battery_pct` | integer | Optional. 0–100 |
| `tilt`, `roll` | degrees (float) | Optional. Device attitude |

```json
{
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
```

A session and practice actor that issues the `clubhouse` namespace and has paused play:

```json
{
  "kind": "bus.actor",
  "from": "c19e04b7a2d5f318",
  "name": "Clubhouse",
  "frp": "0.3.0",
  "heartbeat": 2.0,
  "roles": ["session", "practice"],
  "namespaces": ["clubhouse"],
  "blockers": ["paused"]
}
```

---

### `bus.sync`

**Occurrence, any actor.** "Send me the current state." No fields. Sent by an actor when it connects or reconnects.
Every actor that receives it republishes all of its state, `bus.actor` first (see Rehydration).

```json
{ "kind": "bus.sync", "from": "e4a7136b9f02cd58" }
```

---

### `bus.reply`

**Occurrence, the owner of a command.** The answer to a command, sent after the owner has published any state the
command changed.

| Field | Type | Description |
|-------|------|-------------|
| `to` | string | Required. The `from` of the command |
| `command` | string | Required. The command's kind |
| `ref` | string | Required. The command's `ref` |
| `ok` | bool | Required |
| `error` | string | When `ok` is false, one of the values below |
| `blockers` | array of string | With `not_ready`: the owner's own blockers |
| `message` | string | Optional. Text for people |

| `error` | Meaning |
|---------|---------|
| `bad_request` | Invalid fields |
| `not_allowed` | The owner refuses, e.g. a club outside the golfer's bag |
| `not_ready` | The owner can't now; see `blockers` |
| `unsupported` | The owner doesn't do this |

Everyone receives the reply; only `to` acts on it. A successful reply follows the changed state, so the requester
already holds the new state when it reads `ok`.

```json
{
  "kind": "bus.reply",
  "from": "c19e04b7a2d5f318",
  "to": "5b80e2f7d14c9a63",
  "command": "golfer.select",
  "ref": "sel-118",
  "ok": true
}
```

---

### `bus.alert`

**Occurrence, any actor.** A warning, error or critical condition.

| Field | Type | Description |
|-------|------|-------------|
| `severity` | string | Required. `warn`, `error` or `critical` (the sender can't continue and may go away) |
| `message` | string | Required. Text for people |

Informational only; no receiver behavior is required.

```json
{
  "kind": "bus.alert",
  "from": "7d2f9a41c03be856",
  "severity": "warn",
  "message": "Signal weak — reposition device"
}
```

---

## Shot Messages (`shot.*`)

### Shot Lifecycle

Each shot produces a sequence of correlated messages sharing a `shot_id`. Collectors use the **ShotAccumulator**
pattern: collect messages until `shot.finished`, then process the composed shot.

```
shot.trigger
    ↓
shot.launch    ┐
shot.club      │
shot.impact    ├── any order, any may be absent
shot.flight    ┘
    ↓
shot.finished
```

`shot.launch`, `shot.club`, `shot.impact` and `shot.flight` may arrive in any order. Any of them may not arrive at all
(hardware-dependent). `shot.finished` is the launch monitor's last message for the shot; another actor's addendum,
such as a flight model's `shot.flight`, may follow it (see `shot.finished`). A launch monitor sends each of its
lifecycle messages once per shot, with all the data it has for that part of the shot.

A shot is identified by `key.shot_id` alone, so a collector accumulates by `shot_id`, whoever sent each part. All
shot messages come from the launch monitor that saw the strike, except `shot.flight`, which any flight model may also
send (see `shot.flight`).

**Golfer and club are not on shots.** A launch monitor doesn't own them. A collector takes them from the
`golfer.turn` current when `shot.trigger` arrived, and the block from the golfer's open `practice.block`.

**Key.** Every shot message except `shot.raw` carries `key`; on `shot.raw` it is optional.

| Field | Type | Description |
|-------|------|-------------|
| `shot_id` | string (UUID v4) | Required. Globally unique across sessions. Used for machine correlation. The launch monitor that detected the strike generates it; every other actor that sends a message about the shot uses it unchanged |
| `shot_number` | integer | Required. Monotonic counter from the launch monitor, for human display only. When or whether to reset it is up to the launch monitor |

An actor that publishes a message for another actor's shot copies the shot's `key` unchanged.

---

### `shot.trigger`

**Occurrence, launch monitor.** Ball strike detected. Sent immediately; no measurement data is available yet.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |
| `timestamp` | timestamp | Optional. When the launch monitor detected the strike |

```json
{
  "kind": "shot.trigger",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 },
  "timestamp": 1794063845123
}
```

The example timestamp is 2026-11-07T15:04:05.123Z.

`timestamp` comes from the launch monitor's clock. Clocks on different devices drift and may be set wrong, so a
receiver that compares timestamps, e.g. to recognize one strike reported by two launch monitors, allows a tolerance
of its own choosing. `timestamp` is the sender's declaration, like every other field: a receiver does not rely on it
for security. A receiver ignores a `timestamp` that is not a non-negative integer and still processes the trigger.

---

### `shot.launch`

**Occurrence, launch monitor.** Ball launch measurement data.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |
| `launch` | object | Required. Launch fields, below |

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

```json
{
  "kind": "shot.launch",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 },
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
```

`spin_axis` is omitted when `total_spin_rpm` is 0.

`launch` carries launch values only, and every value in it is measured (see Measured Values and Flight Results).
Carry, total distance and the other results of the ball's flight travel in `shot.flight`. In the example, the launch
monitor measured the spin components and translated them into the total and axis.

**Spin.** `backspin_rpm` + `sidespin_rpm` and `total_spin_rpm` + `spin_axis` are two descriptions of the same spin:

```
backspin_rpm   = total_spin_rpm × cos(spin_axis)
sidespin_rpm   = total_spin_rpm × sin(spin_axis)
total_spin_rpm = √(backspin_rpm² + sidespin_rpm²)
spin_axis      = atan2(sidespin_rpm, backspin_rpm)
```

- A launch monitor sends the pair it measures, and should also send the other pair, translated from it as above.
  Many receivers read only the components.
- A translated pair is measured, like the pair it comes from.
- A launch monitor translates one pair only from both values of the other pair; a `total_spin_rpm` of 0 needs no
  axis. A launch monitor that measures only part of a pair sends only what it has: backspin without sidespin means
  `backspin_rpm` alone, and a total spin without an axis means `total_spin_rpm` alone, with no `spin_axis`.
- When both pairs are present, they must agree to within rounding.
- A receiver that receives one pair may compute the other.

---

### `shot.club`

**Occurrence, launch monitor.** Club head measurement data.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |
| `club` | object | Required. Club fields, below |

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
| `club_offset` | distance | Lateral position of club head at impact relative to ball center. Positive = toward toe side |
| `club_height` | distance | Vertical position of club head at impact relative to ball center. Positive = above center |
| `spin_loft` | degrees (float) | Angle between the club head's direction of travel and the face normal at impact, in three dimensions |
| `low_point_distance` | distance | Position of the lowest point of the club head's arc along the target line, relative to the ball. Positive = target side of the ball, negative = behind the ball |

```json
{
  "kind": "shot.club",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 },
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
```

Launch monitors send `spin_loft` only as a measured value. `dynamic_loft − attack_angle` approximates it but ignores
club path and face angle; that approximation is left to receivers (see Derived Values).

---

### `shot.impact`

**Occurrence, launch monitor.** Club face impact location: where on the club face the ball was struck. Not all
hardware can measure face impact; receivers must handle its absence.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |
| `impact` | object | Required. Impact fields, below |

**Impact fields** — all optional, send what the hardware supports:

| Field | Type | Description |
|-------|------|-------------|
| `lateral` | distance | Horizontal offset from face center. Positive = toward toe |
| `vertical` | distance | Vertical offset from face center. Positive = above center |

```json
{
  "kind": "shot.impact",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 },
  "impact": { "lateral": "0.31in", "vertical": "0.15in" }
}
```

---

### `shot.flight`

**Occurrence, launch monitor or any flight model.** Results of the ball's flight: where it landed, where it came to
rest, and how it flew.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |
| `flight` | object | Required. Flight fields, below |

**Flight fields:**

| Field | Type | Description |
|-------|------|-------------|
| `basis` | string | Required. `"tracked"` or `"modeled"`: how the values were obtained (see below) |
| `model` | string | Optional. Free-text name of the flight model and its producer |
| `carry_distance` | distance | Carry distance (ball in air only) |
| `carry_offline` | distance | Lateral offset of the landing point from the target line. Positive = right of target |
| `total_distance` | distance | Total distance including roll |
| `total_offline` | distance | Lateral offset of the rest position from the target line. Positive = right of target |
| `roll_distance` | distance | Roll distance after landing |
| `max_height` | distance | Maximum apex height |
| `flight_time` | float (seconds) | Time of flight (launch to landing) |
| `descent_angle` | degrees (float) | Angle of descent below horizontal at landing |
| `assumed` | object | Optional, `modeled` only. Launch values the flight model assumed because the shot's `launch` lacks them (see below) |

```json
{
  "kind": "shot.flight",
  "from": "e4a7136b9f02cd58",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 },
  "flight": {
    "basis": "modeled",
    "model": "Fairway Physics 1.3",
    "carry_distance": "181.2m",
    "carry_offline": "-7.1m",
    "total_distance": "196.4m",
    "max_height": "28.0m"
  }
}
```

All fields except `basis` are optional; send what the sender has. `carry_offline` and `descent_angle` describe the
landing of a ball in flight; senders omit them for a ball that does not leave the ground (e.g. a putt).

**Several per shot.** Any actor may publish a `shot.flight` for any shot it saw launched; flights are told apart by
`from` and `flight.model`. A launch monitor sends at most one of its own. A collector keeps every flight it receives;
which one it shows is its choice (e.g. prefer `tracked`). Flight models send as soon as they can, also after the
shot's `shot.finished` (see `shot.finished`).

**Basis.** Every `shot.flight` says how its values were obtained:

| Value | Meaning |
|-------|---------|
| `tracked` | The launch monitor followed the ball's flight, and the values come from that tracking |
| `modeled` | A flight model produced the values, e.g. from the launch values |

A result that a flight model completed or extended, e.g. a tracked flight with a modeled landing and roll, is
`modeled`. A launch monitor's own flight model counts as a flight model: carry that a camera unit computes internally
from the launch it measured is `modeled`.

`model` names the flight model and who produced it, e.g. `"Eagle One flight model 2.1 (Birdie Labs)"`. It is free
text for display and logging; receivers do not parse it.

A receiver ignores a `shot.flight` whose `basis` is missing or is not one of the values above.

**Assumed launch values.** A flight model needs a complete launch. When the shot's `launch` lacks a launch value the
model needed, e.g. the spin of a short chip from hardware that measured its speed and angles but not its spin, the
model assumes one. A `modeled` flight may carry those inputs in `assumed`, so that a receiver that flies the ball
itself can use the same values:

```json
{
  "kind": "shot.launch",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "6f1c2a9e-3b7d-4e0a-9c58-1d2e3f4a5b6c", "shot_number": 43 },
  "launch": { "launch_speed": "22.4mps", "launch_azimuth": 0.8, "launch_elevation": 31.5 }
}
```

```json
{
  "kind": "shot.flight",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "6f1c2a9e-3b7d-4e0a-9c58-1d2e3f4a5b6c", "shot_number": 43 },
  "flight": {
    "basis": "modeled",
    "model": "Eagle One flight model 2.1 (Birdie Labs)",
    "carry_distance": "31.2m",
    "carry_offline": "0.4m",
    "total_distance": "36.0m",
    "max_height": "5.1m",
    "assumed": { "backspin_rpm": 4500, "sidespin_rpm": 0 }
  }
}
```

- `assumed` holds any of the launch fields (`launch_speed`, `launch_azimuth`, `launch_elevation`, `backspin_rpm`,
  `sidespin_rpm`, `total_spin_rpm`, `spin_axis`), with the same units and signs as in `launch`. A sender omits
  `assumed` when it has nothing in it.
- An assumed value is not measured, so it never appears in `launch`. `assumed` holds only inputs the model assumed
  because the shot's `launch` lacks that field: a sender does not put a field in `assumed` that the shot's `launch`
  carries.
- The spin rules of `launch` apply to the launch that `launch` and `assumed` describe together (see Spin): `assumed`
  may hold a partial pair, and when both pairs are complete they agree to within rounding.
- An assumed value is one the model used. An assumed 0, such as the sidespin in the example, is the model's input,
  not a placeholder.
- `assumed` belongs to a `modeled` flight only. A receiver ignores `assumed` in a `tracked` flight and uses the rest
  of that flight as usual.
- A receiver that finds a field in both `launch` and `assumed` uses the `launch` value and ignores the assumed copy.
- A receiver that flies the ball from the launch values uses each `launch` value, and fills a value `launch` lacks
  from `assumed` of a `modeled` flight of its choosing.

Measured-only receivers ignore `shot.flight`, and with it any `assumed` values (see Measured Values and Flight
Results).

---

### `shot.finished`

**Occurrence, launch monitor.** The launch monitor has sent everything it has for the shot. Collectors finalize and
emit the accumulated shot.

| Field | Type | Description |
|-------|------|-------------|
| `key` | object | Required. See Key |

```json
{
  "kind": "shot.finished",
  "from": "7d2f9a41c03be856",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 }
}
```

`shot.finished` ends only its sender's messages for the shot. An actor that sends a lifecycle message for a shot
after its own `shot.finished` for that shot is out of order, and a collector logs a warning. Any other actor that adds
to the shot (e.g. a flight model's `shot.flight`) may publish its addendum at any time, before or after the
`shot.finished`, keyed by the shot's `key`. How a collector correlates and shows an addendum is its choice.

---

### `shot.raw`

**Occurrence, launch monitor.** An optional debugging aid: one protocol message, as the launch monitor's
implementation framed it. An actor that speaks another protocol to its hardware, or to its host, can show the
protocol messages behind its FRP messages. FRP treats their content as opaque.

| Field | Type | Description |
|-------|------|-------------|
| `dir` | string | Required. Direction, seen from the actor's implementation: `"rx"` for a message it received, e.g. from the hardware; `"tx"` for a message it sent, e.g. to the hardware |
| `seq` | integer | Required. Per-actor counter: increases by one for every raw frame the actor sends |
| `timestamp` | timestamp | Optional. When the implementation received or sent the message |
| `label` | string | Optional. Short text the implementation chooses, e.g. the message's name |
| `data` | string | Required. The message's bytes as lowercase hexadecimal, two digits per byte |
| `key` | object | Optional. The shot the message belongs to, when the actor knows it |

```json
{
  "kind": "shot.raw",
  "from": "7d2f9a41c03be856",
  "dir": "rx",
  "seq": 1042,
  "timestamp": 1794063845123,
  "label": "result",
  "data": "0a1b2c",
  "key": { "shot_id": "550e8400-e29b-41d4-a716-446655440000", "shot_number": 42 }
}
```

- **Off by default.** An actor sends `shot.raw` only when its own configuration turns raw frames on. A hub may drop
  them instead of forwarding them; one that forwards them passes each unchanged.
- **One message per frame.** `data` holds exactly one protocol message, framed as the implementation frames it. FRP
  describes no format for it, and a receiver does not rely on its content to process shots.
- **Identity.** An actor does not send a raw frame that carries device or user identity, such as a serial number, a
  hardware address, a network name or account data. It applies its own exclusion before sending; a frame it is
  unsure of is not sent.
- **Order.** An actor sends raw frames in the order it received or sent them relative to the messages it builds from
  them: a frame comes before the shot message built from it.
- **Sequence.** `seq` orders an actor's raw frames and lets a forwarder or receiver notice missing ones for
  debugging; nothing requests them again. An actor keeps counting for as long as it uses the
  same actor id, across connections where it can.
- **Not part of the lifecycle.** A raw frame is not a shot lifecycle message, even when it carries a `key`. It does
  not create, complete or extend a shot, does not count toward first message wins or the accumulator timeout, and is
  not ignored or affected by a closed `shot_id`.
- A receiver ignores a `shot.raw` whose `dir`, `seq` or `data` is missing or invalid, including a `data` that is not
  an even number of hexadecimal digits. It ignores a `timestamp` that is not a non-negative integer, a `label` that
  is not a string and a `key` that is not a valid shot key, and still uses the frame. A receiver that does not use
  raw frames ignores them.

---

### ShotAccumulator Pattern

Collectors must not assume that `shot.launch`, `shot.club`, `shot.impact` and `shot.flight` arrive in a fixed order,
or that any of them arrive at all. The recommended pattern:

```
on shot.trigger:
    if shot_id is closed or already has an accumulator: ignore
    create accumulator keyed by shot_id
    record the current golfer.turn and the golfer's open practice.block

on shot.launch / shot.club / shot.impact:
    if accumulator[shot_id] already has that part: ignore
    store it in accumulator[shot_id]

on shot.flight:
    if basis is missing or unrecognized: ignore
    add it to accumulator[shot_id]'s flights

on shot.finished:
    finalize accumulator[shot_id]
    emit composed shot (launch + club + impact + flights, any may be absent)
    mark shot_id finished

on accumulator timeout (no shot.finished within T of shot.trigger):
    discard accumulator[shot_id] without emitting it
    mark shot_id closed

on shot.launch / shot.club / shot.impact / shot.flight for a finished shot_id:
    if it comes from the actor that sent the shot.finished: log a warning
    correlate it with the composed shot as the collector chooses (e.g. update the shot as above)

on any other lifecycle message for a closed or finished shot_id:
    ignore
```

**First message wins.** A collector keeps the first `shot.launch`, `shot.club` and `shot.impact` of a shot and
ignores a repeat. It keeps every `shot.flight`.

Storing each `flight` object whole keeps its `basis`, `model` and `assumed` with the values they describe. A
collector that flies the ball itself reads the launch from the composed shot: each `launch` value, with a missing one
filled from `assumed` of a `modeled` flight of its choosing. A measured-only collector skips `shot.flight` (see Measured Values and
Flight Results).

**Accumulator timeout.** A collector discards an accumulator whose `shot.finished` has not arrived within its
accumulator timeout `T` of the `shot.trigger`, and does not emit the discarded shot. The recommended default is 30
seconds. Some launch monitors deliver their full flight data several seconds after the strike, so a much shorter
timeout discards shots that would have completed. A collector may choose another value.

**Closed shots.** A `shot_id` is closed once its shot has been discarded. A collector ignores every message for a
closed `shot_id` and does not start a new partial shot for it. A collector remembers a finished or closed `shot_id`
for at least `T`.

---

### Derived Values

Receivers commonly display values that follow exactly from FRP fields. FRP does not carry them as fields. This
section is informative; it fixes the formulas and signs so receivers agree.

| Value | Formula | Sign |
|-------|---------|------|
| Face to path | `face_angle − path` | Positive = face points right of the club path |
| Curve | `carry_offline − carry_distance × tan(launch_azimuth)` | Positive = ball curved rightward in flight |
| Roll offline | `total_offline − carry_offline` | Positive = ball rolled rightward |
| Spin loft (approximate) | `dynamic_loft − attack_angle` | — |

Curve and roll offline use `shot.flight` values, together with `shot.launch` values where the formula needs them.
Distances are converted to a common unit before they are combined.

---

### Measured Values and Flight Results

Values in `launch`, `club` and `impact` are measured. Values that need an aero or flight model travel only in
`shot.flight`, whose `basis` says how they were obtained. `source` in a launch monitor's `bus.actor` `device` says
where its shots originate. Together they let a receiver tell a launch monitor's readings apart from modeled results,
and measured shots apart from virtual ones.

**Measured values.** A measured value is one the launch monitor measured, or a simple translation of one:
- converting between the two spin pairs (see Spin);
- converting a unit, precision or sign convention;
- applying a calibration offset, e.g. an impact height corrected for the club in use.

A value that needs no aero or flight model to produce is measured. A value that needs one is not, even when the
launch monitor itself produces it: it is a flight result, and travels in `shot.flight`. A launch value that a flight
model assumed, e.g. a spin estimated from the launch speed and angle, travels only in `assumed` of a `modeled`
flight (see Assumed launch values), never in `launch`. Any other modeled value that has no `shot.flight` field is not
sent.

**Placeholders are omitted.** A sender never sends a value it does not have. It omits the field instead of sending a
placeholder, such as a `launch_elevation` of 0 from hardware that does not measure it, or a `spin_axis` of 0 when the
axis is unknown. A receiver can treat every value present as one the sender has.

#### `source`

`source` in `bus.actor` `device` declares where the actor's shots originate:

| Value | Meaning |
|-------|---------|
| `measured` | A physical launch monitor measured the shots |
| `virtual` | No physical measurement: simulators, mock devices, generated or test shots |

- `source` describes origin only. Relaying a message, through any number of hubs, never changes it.
- `source` describes one actor, not the connection. A host may run a measured launch monitor and a virtual one side by
  side, each as its own actor.
- An actor's `source` does not change. A host whose input behind an actor changes source, e.g. a launch monitor
  replaced by a simulator, runs it as a new actor with a new id.
- A receiver uses the `source` in the most recent `bus.actor` it has received from the actor. Until it has received
  one, or when the value is absent or not one of those above, the actor's source is unknown. A receiver that requires
  measured shots treats an unknown source as not measured.
- An actor that takes shots from an application that does not speak FRP declares their origin when it knows it. For
  example, a launch monitor's own companion application reports the launch monitor's measured shots, so an actor
  reading them declares `measured`. When it does not know the origin, it omits `source`. It sends the application's
  measured values in `launch`, `club` and `impact`, and the application's flight results in `shot.flight`, with
  `basis` `modeled` unless it knows the launch monitor tracked them. It omits values the application fills in as
  placeholders.
- An actor that reads a launch monitor's hardware itself declares `measured`. An actor that generates shots declares
  `virtual`.

#### Measured-only receivers (informative)

A receiver that needs measured data only, e.g. to record a tournament round:
- drops every shot from a launch monitor whose `source` is not `measured`, including an unknown source;
- ignores `shot.flight` entirely, whatever its `basis` or sender. It therefore never sees `assumed` launch values,
  which travel only there.

What the receiver does with the remaining values is its own choice; for example, a simulator computes the ball's
flight from the launch values with its own model.

---

## Golfer Messages (`golfer.*`)

### `golfer.profile`

**State, the actor that issues the id's namespace; keyed by `id`.** A golfer known to that actor.

| Field | Type | Description |
|-------|------|-------------|
| `id` | golfer id | Required. In a namespace the sender lists in its `bus.actor` `namespaces` |
| `name` | string | Required unless `removed`. Display name |
| `handedness` | handedness | Optional |
| `bag` | array of club code | Optional. The golfer's clubs, in the order the golfer wants them shown |
| `sex` | string | Optional. `"male"` or `"female"`, for published norms that depend on it |
| `height` | distance | Optional |
| `removed` | `true` | The golfer is gone |

Linking two ids that are one person (e.g. a session golfer and a game profile) is done by collectors, never on the
bus.

```json
{
  "kind": "golfer.profile",
  "from": "c19e04b7a2d5f318",
  "id": "clubhouse.0f8c2d1a-77b4-4e1e-9a50-3c6d2b8e4f10",
  "name": "Sam",
  "handedness": "lh",
  "bag": ["DR", "3W", "5H", "6I", "7I", "8I", "9I", "PW", "SW", "PT"],
  "height": "175cm"
}
```

---

### `golfer.turn`

**State, `session`.** Who is up and with what. Published whenever any field changes. Launch monitors read
`handedness`, `club` and `mode` as tracking hints.

| Field | Type | Description |
|-------|------|-------------|
| `golfer` | golfer id or null | Required. Any namespace; the session needn't know the profile. Null: no golfer selected |
| `club` | club code or null | Required. Null: no club (unknown or not applicable) |
| `handedness` | handedness | Required. The session's own value (from its golfer record, the game, or the user) |
| `mode` | mode | Required |

```json
{
  "kind": "golfer.turn",
  "from": "c19e04b7a2d5f318",
  "golfer": "backnine.local-3",
  "club": "7I",
  "handedness": "rh",
  "mode": "full"
}
```

**Tracking hints.** Radar launch monitors typically have distinct arming modes for full swings, chips and putts, each
optimized for different ball speeds and trajectories; some tune detection to the golfer's handedness or to the club
in play (a unit may load a per-club profile before the shot, or arm only when the club is a putter).

- Every `golfer.turn` is complete: a receiver never merges it with an earlier one.
- A launch monitor applies each field it supports and ignores the rest, and still applies the other fields when it
  ignores one.
- A launch monitor treats a null or unknown value in any field (`club`, `handedness` or `mode`) as no hint for that
  field, the same as never having been told. What the hardware does then is the launch monitor's own behavior.
- `club` does not set or imply `mode`: a session that wants putting detection publishes `"mode": "putting"`, with or
  without `"club": "PT"`. If a launch monitor uses both and they conflict, `mode` governs detection.
- A launch monitor that recognizes a club code but has no matching setting may apply the nearest club it supports,
  or ignore the field.
- The hints describe the upcoming shot to the launch monitor; they are not a shot record, and launch monitors do not
  echo them on shot messages.

The session's own readiness (e.g. `paused`) is in its `bus.actor` `blockers`.

---

### `golfer.select`

**Command, to `session`.** Changes the turn. Any subset of the fields; an absent field stays as it is.

| Field | Type | Description |
|-------|------|-------------|
| `ref` | string | Required |
| `golfer` | golfer id or null | Optional. Null clears it |
| `club` | club code or null | Optional. Null: no club |
| `handedness` | handedness | Optional |
| `mode` | mode | Optional |

The session may refuse (`not_allowed`, e.g. the club isn't in the golfer's bag). Whether choosing a golfer also
changes club or handedness is the session's behavior; the resulting `golfer.turn` shows the outcome.

```json
{
  "kind": "golfer.select",
  "from": "5b80e2f7d14c9a63",
  "ref": "sel-118",
  "golfer": "backnine.local-3",
  "handedness": "rh"
}
```

---

## Practice Messages (`practice.*`)

### `practice.block`

**State, `practice`; keyed by `id`.** An open practice block: a run of shots by one golfer. One open block per
golfer. Only open blocks are state; closing a block is its `removed` message, after which it is history kept by the
practice actor. A block carries no shot list.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID v4) | Required |
| `golfer` | golfer id or null | Required unless `removed`. Null: the block for shots with no golfer selected |
| `club` | club code or null | Required unless `removed`. Null: no club (unknown or not applicable) |
| `number` | integer | Required unless `removed`. Counts from 1 within the session, per golfer |
| `session` | integer | Required unless `removed`. The practice session number |
| `started_at` | timestamp | Required unless `removed` |
| `removed` | `true` | The block is closed |

The practice actor rolls a golfer's block when `golfer.turn` changes club (its own rule, not protocol), before the
next shot. A collector puts a shot in the open block for the shot's golfer when `shot.trigger` arrives.

```json
{
  "kind": "practice.block",
  "from": "c19e04b7a2d5f318",
  "id": "9b2e6f04-1c3d-4a8b-b7e5-2f0d9c6a1e33",
  "golfer": "clubhouse.0f8c2d1a-77b4-4e1e-9a50-3c6d2b8e4f10",
  "club": "7I",
  "number": 4,
  "session": 12,
  "started_at": 1794063845123
}
```

---

### `practice.new_block`

**Command, to `practice`.** Starts a new block for a golfer, or a new practice session. Without `new_session`: closes
the golfer's open block and opens a new one for that golfer now (same session).

| Field | Type | Description |
|-------|------|-------------|
| `ref` | string | Required |
| `golfer` | golfer id or null | Optional. Default: the golfer in `golfer.turn`. Ignored with `new_session` |
| `new_session` | bool | Optional, default false. True: close every open block, count `session` up, and open a block for the golfer in `golfer.turn`; other golfers get a block in the new session when they are next up |

```json
{ "kind": "practice.new_block", "from": "c19e04b7a2d5f318", "ref": "nb-7", "new_session": true }
```

---

## Game Messages (`game.*`)

Golfers in game messages are in the game's own namespace. The game publishes `golfer.profile` for them.

### `game.round`

**State, `game`.** The round being played. When the round ends or the game resets, the game sends `status: ended`,
then `removed` when it no longer has a round. When the game removes `game.round`, it also removes `game.ball` and
`game.scores`. What hardware does when a round ends is its own behavior.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Required. The game's round id |
| `status` | string | Required. `loading`, `playing`, `paused` or `ended` |
| `end_reason` | string | With `ended`: `completed` or `abandoned` |
| `course` | object | Optional. `name` (string), `holes` (integer), `par` (integer) |
| `scoring` | string | Optional. `stroke`, `stableford`, `match_play`. Others allowed |
| `players` | array | Required. In playing order; see below |
| `up` | golfer id or null | Required. Whose turn it is in the game |
| `hole` | object or null | Required. The current hole; see below. Null between holes |
| `removed` | `true` | The game no longer has a round |

Each `players` entry: `golfer` (golfer id), `name` (string), `team` (integer, optional), `handicap` (float,
optional), `local` (bool: playing at this simulator).

`hole`: `number`, `par`, `stroke_index` (integers), `length` (distance), `pin` (position).

```json
{
  "kind": "game.round",
  "from": "5b80e2f7d14c9a63",
  "id": "r-20261110-1",
  "status": "playing",
  "course": { "name": "Pine Hollow", "holes": 18, "par": 72 },
  "scoring": "stroke",
  "players": [
    { "golfer": "backnine.local-3", "name": "Sam", "handicap": 12.4, "local": true },
    { "golfer": "backnine.remote-1", "name": "Alex", "local": false }
  ],
  "up": "backnine.local-3",
  "hole": { "number": 7, "par": 4, "stroke_index": 3, "length": "391yd", "pin": ["12.0m", "355.2m", "3.1m"] }
}
```

---

### `game.ball`

**State, `game`.** The ball about to be played: where it lies and whether the game will take a shot now.

| Field | Type | Description |
|-------|------|-------------|
| `golfer` | golfer id | Required |
| `hole` | integer | Required |
| `stroke_number` | integer | Required. The stroke about to be played |
| `allowed_commands` | array of string | Optional. Command kinds the game accepts now, e.g. `["game.aim"]` |
| `lie` | object | Optional. See below |
| `position` | position | Optional |
| `aim` | position or null | Optional. Where the shot is aimed; null when the game aims at its default |
| `distance_to_hole` | distance | Optional |
| `decision` | string | While the game's `bus.actor` lists `awaiting_decision`: text for people (e.g. "Take a drop?") |

`lie`: `surface` (open string: `tee`, `fairway`, `rough`, `sand`, `fringe`, `green`, ...), `slope_up_down` (degrees,
positive = uphill toward the target), `slope_left_right` (degrees, positive = ground rises to the right of the target
line).

```json
{
  "kind": "game.ball",
  "from": "5b80e2f7d14c9a63",
  "golfer": "backnine.local-3",
  "hole": 7,
  "stroke_number": 2,
  "allowed_commands": ["game.aim"],
  "lie": { "surface": "fairway", "slope_up_down": 1.5, "slope_left_right": -0.8 },
  "position": ["8.4m", "212.6m", "1.2m"],
  "aim": null,
  "distance_to_hole": "157yd"
}
```

---

### `game.stroke`

**Occurrence, `game`.** The game's view of one stroke, sent once when the ball is at rest. Field names reuse
`shot.launch` and `shot.flight`; no new units or signs.

| Field | Type | Description |
|-------|------|-------------|
| `stroke_id` | string | Required. The game's id for the stroke |
| `shot_id` | string (UUID v4) | Optional. The launch monitor's `shot_id`; absent for strokes with no shot (gimme, auto putt, penalty only) |
| `replaces` | string | Optional. The `stroke_id` this stroke replaces (mulligan, re-hit) |
| `golfer` | golfer id | Required |
| `hole` | integer | Required |
| `stroke_number` | integer | Required |
| `flown` | object | Optional. The launch the game flew, with launch field names; may differ from what was measured (game assists, putting model) |
| `flight` | object | Optional. Flight field names; `basis` is `modeled` and `model` the game's |
| `outcome` | string | Required. `in_play`, `green`, `holed`, `out_of_bounds`, `water`, `lost`, `unplayable`, `penalty_only`, `gimme` or `auto_putt` |
| `putt` | bool | Optional |
| `penalty_strokes` | integer | Optional, default 0 |
| `lie` | object | Optional. Where the ball came to rest, as in `game.ball` |
| `start_position`, `end_position` | position | Optional |
| `start_distance_to_hole`, `end_distance_to_hole` | distance | Optional |

A stroke counts unless a later stroke `replaces` it, so a mulligan needs no second message for the first stroke.

```json
{
  "kind": "game.stroke",
  "from": "5b80e2f7d14c9a63",
  "stroke_id": "r-20261110-1/7/1",
  "shot_id": "550e8400-e29b-41d4-a716-446655440000",
  "golfer": "backnine.local-3",
  "hole": 7,
  "stroke_number": 1,
  "flown": { "launch_speed": "67.2mps", "launch_azimuth": -1.3, "launch_elevation": 14.2, "backspin_rpm": 3200, "sidespin_rpm": -450 },
  "flight": { "basis": "modeled", "model": "Back Nine Golf", "carry_distance": "198yd", "total_distance": "234yd", "max_height": "31yd" },
  "outcome": "in_play",
  "lie": { "surface": "fairway", "slope_up_down": 1.5, "slope_left_right": -0.8 },
  "start_distance_to_hole": "391yd",
  "end_distance_to_hole": "157yd"
}
```

---

### `game.scores`

**State, `game`.** The scores of the round.

| Field | Type | Description |
|-------|------|-------------|
| `round` | string | Required. The `game.round` `id` |
| `players` | array | Required. One entry per golfer; see below |

Each `players` entry — all fields but `golfer` optional:

| Field | Type | Description |
|-------|------|-------------|
| `golfer` | golfer id | The golfer |
| `strokes` | integer | Total strokes |
| `to_par` | integer | |
| `net_to_par` | integer | |
| `points` | integer | Stableford points |
| `rank` | integer | |
| `holes` | array | Per hole; see below |

Each `holes` entry — all fields but `number` optional: `number`, `strokes`, `putts`, `to_par`, `net_to_par`, `points`
(integers), `result` (match play: `won`, `lost` or `halved`), `holed_out` (bool), `final` (bool: the score won't
change).

```json
{
  "kind": "game.scores",
  "from": "5b80e2f7d14c9a63",
  "round": "r-20261110-1",
  "players": [
    {
      "golfer": "backnine.local-3",
      "strokes": 27,
      "to_par": 3,
      "holes": [
        { "number": 6, "strokes": 4, "putts": 2, "to_par": 0, "holed_out": true, "final": true },
        { "number": 7, "strokes": 1, "holed_out": false, "final": false }
      ]
    }
  ]
}
```

---

### `game.aim`

**Command, to `game`.** Changes where the next shot is aimed.

| Field | Type | Description |
|-------|------|-------------|
| `ref` | string | Required |
| `action` | string | Required. `nudge`, `set` or `reset` |
| `degrees` | float | With `nudge`: −45 to 45, positive = right (target line frame) |
| `target` | position | With `set` |

```json
{ "kind": "game.aim", "from": "c19e04b7a2d5f318", "ref": "aim-3", "action": "nudge", "degrees": -2.0 }
```

---

## Robustness

- Unknown `kind` values must be silently ignored, extension kinds (`x.*`) included
- Unknown fields on known messages must be silently ignored
- Missing optional fields must be handled gracefully
- Invalid JSON must be silently ignored
- Collectors must not crash on partial shots (e.g. `shot.launch` with no `shot.flight`, `shot.club` or
  `shot.impact`)
- Collectors must handle shot messages for unknown `shot_id` values gracefully (e.g. if the collector connected
  mid-shot and missed the `shot.trigger`)
- Receivers must handle an actor going away at any time, with or without a preceding `bus.alert` or `removed`

---

## Versioning

`frp` in `bus.actor` states the protocol version the actor implements. The version follows **semver**. Breaking
changes increment the major version, or the minor version during the 0.x draft. Adding an optional field or a new kind
is not a breaking change: receivers ignore unknown kinds and fields and handle missing optional fields (see
Robustness), and the version string stays the same.

`frp` is informational: a receiver may show a mismatch; no other behavior is required.

Version strings are plain semver (e.g. `"0.3.0"`). They carry no draft, release-candidate or status suffix.

---

## Relationship to Other Standards

**[Render Relay Protocol (RRP)](https://github.com/renderrelay/spec)** `golf.frp` extension tunnels FRP messages over
an RRP WebSocket connection. FRP is the source of truth for all message definitions.

---

## Compliance

**Every compliant actor** must:
- Send `kind` and its actor id as `from` on every message
- Publish `bus.actor` at start, on any change, in answer to `bus.sync`, and every `heartbeat` seconds
- Send `bus.sync` when it connects or reconnects, and republish all of its state, `bus.actor` first, when it receives
  one
- Publish state only for what it owns, as complete objects, and remove state with a `removed` message
- List only its own blockers in `blockers`
- Treat an actor it has not heard a `bus.actor` from for three heartbeats as gone, and forget all its state
- Ignore unknown kinds and unknown fields
- Use unit-tagged strings for all velocity and distance fields, and omit every value it does not have rather than
  sending a placeholder

**An owner** (`session`, `practice`, `game`, or an actor issuing a namespace) must:
- Publish its state when it changes, at start, and in answer to `bus.sync`
- Answer every command of a kind it owns with `bus.reply`, after publishing any state the command changed

**A compliant launch monitor** must:
- Send `shot.trigger` immediately on ball strike detection
- Send `shot.finished` to terminate every shot
- Send only measured values in `launch`, `club` and `impact`, and values that need a flight model only in
  `shot.flight`, with its `basis`
- Send at most one `shot.flight` of its own per shot
- Send `assumed` only in a `modeled` flight, and only for launch fields the shot's `launch` does not carry
- Apply the `golfer.turn` fields it supports and ignore the rest
- Send `shot.raw` only when its own configuration turns raw frames on, never send a raw frame that carries device or
  user identity, increase `seq` by one for each raw frame it sends, and send `data` as lowercase hexadecimal of
  exactly one message

A compliant launch monitor should:
- Send each shot lifecycle message at most once per shot
- Send `timestamp` on `shot.trigger` when it has a clock set to UTC
- Send the other spin pair, translated from the pair it measures
- Name its flight model in `model` when it sends a `modeled` flight
- Declare `source` in its `bus.actor` `device` when it knows the origin of its shots

**A compliant flight model** must send `shot.flight` with `basis` `modeled`, and `assumed` only for launch fields the
shot's `launch` does not carry. It should name its model in `model` and send as soon as it can, before or after the
shot's `shot.finished`.

**A compliant collector** must:
- Implement the ShotAccumulator pattern, including the accumulator timeout: discard a shot whose `shot.finished` does
  not arrive in time, and ignore messages for a closed `shot_id`
- Handle missing `shot.launch`, `shot.club`, `shot.impact` or `shot.flight` gracefully
- Keep only the first `shot.launch`, `shot.club` and `shot.impact` of a shot, and keep every `shot.flight`
- Ignore a `shot.flight` whose `basis` is missing or unrecognized
- Ignore `assumed` in a `tracked` flight, and use the `launch` value of a field that is in both `launch` and `assumed`
- Ignore a `timestamp` that is not a non-negative integer, and still process the trigger
- Log a warning when an actor sends a lifecycle message for a shot after its own `shot.finished` for that shot, and
  not for another actor's addendum
- If it requires measured shots, treat a launch monitor whose `source` is absent or unrecognized as not measured, and
  ignore `shot.flight`
- Keep `shot.raw` out of the ShotAccumulator: raw frames never create, complete or change a shot

A compliant collector should use an accumulator timeout of 30 seconds unless it has a reason to choose another.

**A compliant hub** must forward each message it relays unchanged to every other connection and never back to the one
it came from. Which kinds it relays over a connection is its own configuration (default: all).

---

## License

The Flight Relay Protocol specification is released under
[CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — no rights reserved. Implement it freely, in
any product, open or closed, without attribution.

---

## Appendix: Changes from 0.2.0

0.3.0 is a breaking revision. An endpoint implements one major/minor version, so a 0.3.0 actor and a 0.2.0 endpoint
do not interoperate.

- **Bus model.** The device and controller roles, the `start`/`init` handshake and the `device` + `event` envelope
  are replaced by actors on a bus with a flat envelope (`kind`, `from`), ownership, `bus.sync` and heartbeats.
- **Renamed kinds.** `shot_trigger`, `launch`, `club_path`, `face_impact`, `flight_result`, `shot_finished`,
  `raw_frame` and `alert` are `shot.trigger`, `shot.launch`, `shot.club`, `shot.impact`, `shot.flight`,
  `shot.finished`, `shot.raw` and `bus.alert`. Fields inside `key`, `launch`, `club`, `impact` and `flight` are
  unchanged.
- **`device_telemetry`** is replaced by `bus.actor`, with typed `device` values. The `ready` key is replaced by
  `blockers`.
- **`set_detection_mode`** is replaced by `golfer.turn` (owned by the session) and `golfer.select`. `handed` is
  `handedness`. Every `golfer.turn` is complete and a receiver never merges it with an earlier one: a null `club` is
  no club, and a null or unknown value in any field is no hint for that field.
- **`shot.flight`**: several per shot, from any flight model, which may send it after the shot's `shot.finished`. A
  hub no longer adds or deduplicates flights.
- **Raw frames** are turned on by the sending actor's own configuration.
- **Forwarding** duplicate detection is removed; hubs forward every message, and loops are a configuration error.
- **New kinds:** `bus.sync`, `bus.reply`, `golfer.profile`, `golfer.turn`, `golfer.select`, `practice.block`,
  `practice.new_block`, `game.round`, `game.ball`, `game.stroke`, `game.scores`, `game.aim`.

---

*FRP 0.3.0 — [github.com/flightrelay/spec](https://github.com/flightrelay/spec)*
