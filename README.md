# Flight Relay Protocol (FRP)

**The open standard for golf launch monitor event streaming.**

[Read the spec →](SPEC.md)

---

## The Problem

Every golf launch monitor speaks a different language.

Foresight, Garmin, Uneekor, Rapsodo, Bushnell — each has a proprietary protocol, a proprietary SDK, and proprietary terms around how third parties can integrate with their hardware. Open source golf simulators that want to support multiple launch monitors have two options: negotiate SDK access with every vendor individually, or reverse engineer the protocols themselves.

Neither works well.

SDK agreements take time, create legal overhead, and are often unavailable to open source projects entirely. Reverse engineering produces fragile integrations that break on firmware updates, vary in quality across devices, and create a permanently fragmented ecosystem where popular monitors get decent support and everything else gets none.

The result is that the open source golf sim ecosystem — which should be the most interoperable space in the sport — is instead balkanized by proprietary hardware boundaries. Golfers with less common monitors are left out. Developers spend months on hardware compatibility instead of simulator features.

## The Solution

Flight Relay Protocol (FRP) is a minimal open standard for streaming launch monitor shot data over WebSocket.

It defines the events that matter — shot trigger, launch, flight result, club path, face impact, shot finished, device telemetry, alerts, and detection mode commands — plus optional raw frames for debugging. Any launch monitor vendor can implement a small FRP device alongside their existing software stack in a weekend. Any open source simulator, stat tracker, video overlay, or analysis tool can implement an FRP controller once and work with every compliant device immediately.

## For Launch Monitor Vendors

Implementing FRP does not mean opening your software stack, sharing your algorithms, or publishing an SDK.

Your firmware, your signal processing, your ball flight modeling, your proprietary app — none of that changes. FRP sits at the output layer: you emit shot events in a standard format over a local WebSocket. What happens inside your device to produce those events is entirely your own.

**What you get:**

- Immediate compatibility with every FRP-compliant open source project, with no ongoing SDK maintenance burden
- No support obligation — the open source community consumes your events, not your codebase
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

FRP is intentionally narrow — it defines the shot events and device telemetry that flow from the launch monitor, plus a small set of commands back (detection mode, with optional handedness and club hints). It does not define transport discovery, authentication, configuration, or device management. Those concerns belong to your project. FRP is easy to embed in whatever architecture you already have.

## How It Works

FRP is a WebSocket protocol. After a brief handshake, the device streams JSON events for every shot lifecycle stage:

```
shot_trigger  →  launch + flight_result + club_path + face_impact (any order)  →  shot_finished
```

Each shot is correlated by a `shot_id`, and `shot_trigger` can carry the time of the strike. Controllers accumulate events until `shot_finished`, then process the composed shot. A shot that doesn't finish within the accumulator timeout (30 seconds recommended) is discarded. All velocity and distance values use unit-tagged strings (`"67.2mps"`, `"180.5m"`, `"197.4yd"`) — no implicit unit assumptions, no conversion bugs. Angles are decimal degrees, spin is RPM, time is seconds.

```javascript
ws.send(
  JSON.stringify({ kind: "start", version: ["0.2.0"], name: "My Dashboard" }),
);

const shots = {};

// Launch values for flying the ball yourself: each measured value, with a
// missing one filled from what a modeled flight result assumed.
const launchForFlight = (shot) => ({
  ...(shot.flight?.basis === "modeled" ? shot.flight.assumed : {}),
  ...shot.launch,
});

ws.onmessage = (msg) => {
  const message = JSON.parse(msg.data);
  const event = message.event;
  if (!event || !event.key) return; // handle device_telemetry, alert separately

  const id = event.key.shot_id;

  // A shot keeps the first event of each kind.
  if (event.kind === "shot_trigger") shots[id] ??= {};
  if (event.kind === "launch") shots[id].launch ??= event.launch;
  if (event.kind === "flight_result") shots[id].flight ??= event.flight; // also check basis is "tracked" or "modeled"
  if (event.kind === "club_path") shots[id].club ??= event.club;
  if (event.kind === "face_impact") shots[id].impact ??= event.impact;
  if (event.kind === "shot_finished") (process(shots[id]), delete shots[id]);
};
```

That's the core of an FRP controller. The full spec is one markdown file.

## Why Multiple Events Per Shot?

Launch monitors don't produce all their data at once. Ball launch data (speed, angles, spin) is typically available within milliseconds of impact from radar processing, while carry and the rest of the flight result may come seconds later from tracking the ball or from a flight model. Club data may take longer — some devices use separate sensors or processing pipelines. Face impact location, when available, requires fusing radar and camera data with significant post-processing delay.

FRP splits the shot into separate events so devices can emit each measurement **as soon as it's ready**, rather than blocking on the slowest pipeline. A simulator can start animating the ball flight immediately on `launch`, then incorporate club and face impact data when they arrive. `shot_finished` signals that all available data for the shot has been sent.

This design means controllers never wait longer than necessary, and devices never need to buffer data they already have.

## Measured or Modeled?

FRP keeps what a launch monitor measured apart from what a flight model produced. `launch` carries the launch only (ball speed, launch angles and spin), and everything in it is measured. Carry, total distance, apex and the rest of the flight go in `flight_result`, whose required `basis` says whether the device tracked the ball's flight (`tracked`) or a flight model produced the values (`modeled`). When the launch lacks a value the model needed, such as the spin of a chip the device couldn't read, a `modeled` result lists what the model assumed in `assumed`; it never appears in `launch`. A device never sends a placeholder for a value it doesn't have. `device_telemetry` declares the device's `source`: `measured` (a physical launch monitor measured the shots) or `virtual` (simulated or generated shots). Relaying a shot never changes its source.

A controller that needs measured data only, such as one recording a tournament round, drops shots whose source isn't `measured` and ignores `flight_result`, assumed values included, then flies the ball with its own model. Everyone else can use both.

## Bridges and Forwarding

A bridge or hub that forwards events keeps each shot's `device`, `shot_id`, `shot_number` and `timestamp` unchanged, so a controller can recognize the same shot arriving by two routes. A forwarder may add a `flight_result` from its own model to a shot whose device sent none, marked `modeled`, naming the model and listing any launch values it assumed. Every endpoint that forwards events drops duplicates first: a shot event already forwarded, telemetry that changes nothing, an alert already sent, a raw frame whose sequence number it has seen. A repeat stops at the first hop, so an event that loops back dies out. How forwarders form a bus and find each other is outside the spec.

## Raw Frames for Debugging

A device that translates another protocol into FRP can also show what it saw. A controller that sends `"raw_frames": true` in `start` gets a `raw_frame` event for each protocol message the device received (`rx`) or sent (`tx`): the message as hex, a per-device sequence number, an optional timestamp and label, and the shot it belongs to when the device knows it. Raw frames are off unless asked for, never carry device or user identity such as serial numbers or network names, and never affect a shot. FRP treats their content as opaque.

## Ecosystem

FRP is the primitive layer of a broader open standards stack for golf simulation:

**[Render Relay Protocol (RRP)](https://github.com/renderrelay/spec)** tunnels FRP events and commands over its `golf.frp` extension, carrying FRP fields inside the RRP `ext` envelope. Any FRP-compatible launch monitor or bridge can relay shot data through an RRP renderer to any RRP viewer — no direct integration required, and FRP remains the single source of truth for event definitions.

**[Flighthook](https://github.com/divotmaker/flighthook)** uses FRP as its core event model, extending it with a multi-device actor system, REST API, unit conversion, and configuration management. For projects that need more than raw shot events, Flighthook is an integration layer built on FRP.

Together these standards enable a fully open golf simulation stack:

```
Launch Monitor
    ──FRP──▶ Golf Simulator / RRP Renderer
        ──RRP──▶ Any RRP viewer (Roku, Apple TV, browser)
                 (shot data relayed via golf.frp extension)
```

## Status

FRP is in early draft. The spec is open for feedback. The current version is 0.2.0, a breaking revision of 0.1.0: `ball_flight` is replaced by `launch`, which carries launch values only, and flight results moved to the new `flight_result`.

- [x] Protocol spec draft
- [x] Reference implementation (Rust, via [Flighthook](https://github.com/divotmaker/flighthook))
- [ ] Conformance test suite

## SDKs

| Language | Repository | Status |
| -------- | ---------- | ------ |
| Rust | [`flightrelay/sdk-rust`](https://github.com/flightrelay/sdk-rust) | Available |

## License

The Flight Relay Protocol specification is [CC0](LICENSE) — no rights reserved. Implement it freely, in any product, open or closed, without attribution.
