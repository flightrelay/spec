# Flight Relay Protocol (FRP)

**The open bus for golf simulation data.**

[Read the spec →](SPEC.md)

---

## The Problem

Every golf launch monitor speaks a different language.

Foresight, Garmin, Uneekor, Rapsodo, Bushnell — each has a proprietary protocol, a proprietary SDK, and proprietary terms around how third parties can integrate with their hardware. Open source golf simulators that want to support multiple launch monitors have two options: negotiate SDK access with every vendor individually, or reverse engineer the protocols themselves.

Neither works well.

SDK agreements take time, create legal overhead, and are often unavailable to open source projects entirely. Reverse engineering produces fragile integrations that break on firmware updates, vary in quality across devices, and create a permanently fragmented ecosystem where popular monitors get decent support and everything else gets none.

The result is that the open source golf sim ecosystem — which should be the most interoperable space in the sport — is instead balkanized by proprietary hardware boundaries. Golfers with less common monitors are left out. Developers spend months on hardware compatibility instead of simulator features.

## The Solution

Flight Relay Protocol (FRP) is an open bus for golf simulation data. Launch monitors, sessions, practice tools, games, flight models, lights and collectors are all **actors**: each publishes JSON messages to the bus and receives every message on it. Each piece of state has one owner, a message carries only what its sender owns, consumers do the joining, and an actor that reconnects asks for the current state and carries on from there.

It defines the messages that matter — the shot (trigger, launch, club, face impact, flight, finished), who is up and with what club, practice blocks, the game's round, ball, strokes and scores, plus actor status and a few commands — and optional raw frames for debugging. Any launch monitor vendor can implement a small FRP actor alongside their existing software stack in a weekend. Any open source simulator, stat tracker, video overlay, or analysis tool can implement FRP once and work with every compliant launch monitor immediately.

## For Launch Monitor Vendors

Implementing FRP does not mean opening your software stack, sharing your algorithms, or publishing an SDK.

Your firmware, your signal processing, your ball flight modeling, your proprietary app — none of that changes. FRP sits at the output layer: you publish shot messages in a standard format over a local WebSocket. What happens inside your device to produce those messages is entirely your own.

**What you get:**

- Immediate compatibility with every FRP-compliant open source project, with no ongoing SDK maintenance burden
- No support obligation — the open source community consumes your messages, not your codebase
- No reverse engineering of your hardware — FRP gives the community a clean, supported integration path, which is always preferable to a fragile reverse-engineered one
- A better ecosystem around your hardware, which makes your hardware more valuable

**What you keep:**

- Full control of your software stack
- Full control of your proprietary app and cloud services
- No obligation to document internals, maintain client libraries, or respond to SDK support requests

FRP is CC0 — no rights reserved, no attribution required, no lawyers needed.

## For Open Source Projects

Implement FRP once. Support every compliant launch monitor forever.

No reverse engineering. No fragile unofficial integrations. No per-device maintenance burden. When a new launch monitor ships FRP support, it works with your project on day one.

FRP is intentionally narrow — it defines the messages and who owns each piece of state. Launch monitors read the golfer's handedness, club and shot mode from the session's `golfer.turn` as tracking hints. It does not define discovery, authentication, configuration, or device management. Those concerns belong to your project or the transport. FRP is easy to embed in whatever architecture you already have.

## How It Works

FRP's WebSocket binding has no handshake. On connecting, each actor announces itself with `bus.actor`, republishes its state and asks for the current state with `bus.sync`. Every message is a flat JSON object with a `kind` and the sending actor's id in `from`. A launch monitor publishes each stage of a shot as soon as it has it:

```
shot.trigger  →  shot.launch + shot.club + shot.impact + shot.flight (any order)  →  shot.finished
```

Each shot is correlated by a `shot_id`, and `shot.trigger` can carry the time of the strike. Collectors accumulate messages until `shot.finished`, then process the composed shot. A shot that doesn't finish within the accumulator timeout (30 seconds recommended) is discarded. All velocity and distance values use unit-tagged strings (`"67.2mps"`, `"180.5m"`, `"197.4yd"`) — no implicit unit assumptions, no conversion bugs. Angles are decimal degrees, spin is RPM, time is seconds.

```javascript
const me = loadOrCreateActorId(); // random, generated once, kept across restarts
const actor = { kind: "bus.actor", from: me, name: "My Dashboard", frp: "0.3.0",
                heartbeat: 2.0, roles: ["collector"], blockers: [] };

ws.onopen = () => {
  ws.send(JSON.stringify(actor));
  ws.send(JSON.stringify({ kind: "bus.sync", from: me }));
  setInterval(() => ws.send(JSON.stringify(actor)), actor.heartbeat * 1000);
};

const shots = {};

// Launch values for flying the ball yourself: each measured value, with a
// missing one filled from what a modeled flight assumed.
const launchForFlight = (shot) => ({
  ...(shot.flights.find((f) => f.basis === "modeled")?.assumed ?? {}),
  ...shot.launch,
});

ws.onmessage = (msg) => {
  const m = JSON.parse(msg.data);
  if (m.kind === "bus.sync") return ws.send(JSON.stringify(actor));
  if (!m.kind?.startsWith("shot.") || !m.key) return; // bus.actor, golfer.turn, ... handled separately

  const id = m.key.shot_id;

  // A shot keeps the first launch, club and impact, and every flight.
  if (m.kind === "shot.trigger") shots[id] ??= { flights: [] };
  const shot = shots[id];
  if (!shot) return;
  if (m.kind === "shot.launch") shot.launch ??= m.launch;
  if (m.kind === "shot.club") shot.club ??= m.club;
  if (m.kind === "shot.impact") shot.impact ??= m.impact;
  if (m.kind === "shot.flight") shot.flights.push(m.flight); // check basis is "tracked" or "modeled"
  if (m.kind === "shot.finished") (process(shot), delete shots[id]);
};
```

That's the core of an FRP collector. The full spec is one markdown file.

## Why Multiple Messages Per Shot?

Launch monitors don't produce all their data at once. Ball launch data (speed, angles, spin) is typically available within milliseconds of impact from radar processing, while carry and the rest of the flight result may come seconds later from tracking the ball or from a flight model. Club data may take longer — some devices use separate sensors or processing pipelines. Face impact location, when available, requires fusing radar and camera data with significant post-processing delay.

FRP splits the shot into separate messages so launch monitors can publish each measurement **as soon as it's ready**, rather than blocking on the slowest pipeline. A simulator can start animating the ball flight immediately on `shot.launch`, then incorporate club and face impact data when they arrive. `shot.finished` signals that the launch monitor has sent all the data it has for the shot.

This design means consumers never wait longer than necessary, and launch monitors never need to buffer data they already have.

## Measured or Modeled?

FRP keeps what a launch monitor measured apart from what a flight model produced. `shot.launch` carries the launch only (ball speed, launch angles and spin), and everything in it is measured. Carry, total distance, apex and the rest of the flight go in `shot.flight`, whose required `basis` says whether the launch monitor tracked the ball's flight (`tracked`) or a flight model produced the values (`modeled`). Any flight model on the bus may publish its own `shot.flight` for a shot. When the launch lacks a value the model needed, such as the spin of a chip the launch monitor couldn't read, a `modeled` flight lists what the model assumed in `assumed`; it never appears in `shot.launch`. A sender never sends a placeholder for a value it doesn't have. A launch monitor's `bus.actor` declares its `source`: `measured` (a physical launch monitor measured the shots) or `virtual` (simulated or generated shots). Relaying a message never changes it.

A consumer that needs measured data only, such as one recording a tournament round, drops shots whose source isn't `measured` and ignores `shot.flight`, assumed values included, then flies the ball with its own model. Everyone else can use both.

## Hubs

A hub forwards each message it relays unchanged to every other connection, never back to the one it came from. Which kinds it relays over a connection is its own configuration (default: all). Every message keeps its sender's `from`, and every shot keeps the `shot_id` its launch monitor gave it. Hubs connected in a loop are a configuration error. How hubs find each other is outside the spec.

## Raw Frames for Debugging

An actor that translates another protocol into FRP can also show what it saw. When its own configuration turns them on, it publishes a `shot.raw` message for each protocol message it received (`rx`) or sent (`tx`): the message as hex, a per-actor sequence number, an optional timestamp and label, and the shot it belongs to when the actor knows it. Raw frames are off by default, never carry device or user identity such as serial numbers or network names, and never affect a shot. FRP treats their content as opaque.

## Ecosystem

FRP is the primitive layer of a broader open standards stack for golf simulation:

**[Render Relay Protocol (RRP)](https://github.com/renderrelay/spec)** tunnels FRP messages over its `golf.frp` extension, carrying FRP fields inside the RRP `ext` envelope. Any FRP-compatible launch monitor or hub can relay shot data through an RRP renderer to any RRP viewer — no direct integration required, and FRP remains the single source of truth for message definitions.

**[Flighthook](https://github.com/divotmaker/flighthook)** is an implementation of FRP that hosts launch monitor drivers and outputs to simulators.

Together these standards enable a fully open golf simulation stack:

```
Launch Monitor
    ──FRP──▶ Golf Simulator / RRP Renderer
        ──RRP──▶ Any RRP viewer (Roku, Apple TV, browser)
                 (shot data relayed via golf.frp extension)
```

## Status

FRP is in early draft. The spec is open for feedback. The current version is 0.3.0; see the spec's Changes from 0.2.0.

- [x] Protocol spec draft
- [x] Reference implementation (Rust, via [Flighthook](https://github.com/divotmaker/flighthook))
- [ ] Conformance test suite

## SDKs

| Language | Repository | Status |
| -------- | ---------- | ------ |
| Rust | [`flightrelay/sdk-rust`](https://github.com/flightrelay/sdk-rust) | Available |

## License

The Flight Relay Protocol specification is [CC0](LICENSE) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.
