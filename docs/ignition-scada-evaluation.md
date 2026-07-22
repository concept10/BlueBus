# Evaluation: BlueBus as a Device for Ignition SCADA/HMI

*Goal: demonstrate Inductive Automation Ignition running live in the vehicle
(E53 X5), with BlueBus acting as the field device that bridges the BMW I/K-Bus
into Ignition tags. Ignition-side support would be developed in the
`~/Developer/xign` workspace — see the companion proposal
`xign/docs/OUTLAW_BLUEBUS_MODULE_PROPOSAL.md`.*

## Verdict

**Viable today, with zero firmware changes required for a first demo.**
BlueBus already exposes everything an Ignition gateway needs over its USB
serial port (FT231XS, 115200 baud):

- **Read path** — `SET LOG IBUS ON` streams every I-Bus frame received from
  the vehicle as a timestamped, parseable hex line
  (`LogDebugByteArray`, `firmware/application/lib/ibus.c:1038`).
- **Write path** — the undocumented CLI command `SEND IBUS` transmits
  arbitrary frames onto the I-Bus (`ui/cli.c:751` →
  `IBusSendCommand`, which computes length/checksum). Ignition can therefore
  not just observe the vehicle but act on it: write text to the cluster,
  request sensor status, trigger the same diagnostic jobs the comfort
  features use.

In SCADA terms: BlueBus is a **serial RTU** speaking an ASCII line protocol,
and the I-Bus is the fieldbus behind it. The vehicle becomes a plant floor —
the instrument cluster (IKE) is a PLC broadcasting process values, the light
module and body module are actuators, and PDC is an 8-channel analog sensor
array.

## Demo Architecture

```
   E53 X5 I/K-Bus (9600 baud, single wire)
   IKE ─ GM ─ LCM ─ PDC ─ DSP ─ MFL ─ BMBT ...
        │
   ┌────┴─────┐  USB (FT231XS, 115200)   ┌──────────────────────────┐
   │ BlueBus  ├──────────────────────────┤ In-car gateway computer   │
   │ (CDC 18) │   CLI + I-Bus log stream │ (Pi 5 / NUC / laptop)     │
   └──────────┘                          │ Ignition 8.3 + new module │
                                         │   [BlueBus] tag provider  │
                                         └───────────┬──────────────┘
                                                     │ Wi-Fi AP (local)
                                         ┌───────────┴──────────────┐
                                         │ Tablet / phone           │
                                         │ Perspective session      │
                                         │ (glass dashboard, xign)  │
                                         └──────────────────────────┘
```

- BlueBus stays fully functional as the car's Bluetooth/CD-changer device
  while the gateway listens — logging is passive.
- The gateway computer runs from vehicle 12 V (see Risks for power notes).
- The Perspective client needs no internet: run the gateway as a local Wi-Fi
  access point.

## What BlueBus Provides Today

### Serial interface

| Property | Value |
|---|---|
| Physical | USB via FT231XS (`hardware/`), enumerates as a serial port |
| Baud | 115200, 8 data bits, odd parity for the bootloader; application CLI at 115200 |
| Protocol | Human CLI (`HELP`), line-oriented, `\r\n` terminated |
| Contention | Same port used by the firmware updater — the Ignition module must release the port during updates |

### Read path — I-Bus frame stream

Enable with `SET LOG IBUS ON` (persisted to EEPROM, `ui/cli.c` `SET LOG`
handler). Disable the other sources (`SET LOG BT OFF`, `SYS`, `UI`) to keep
the stream clean. Each received frame arrives as:

```
[<millis>] DEBUG: IBus: RX[<len>]: 80 05 BF 18 3A 2E 5C
```

- `<millis>` — BlueBus uptime timestamp (from `TimerGetMillis()`).
- Hex bytes are the complete raw frame: `src len dst data... checksum`.
- Format defined in `LogDebugByteArray` (`lib/log.c:124`); lines are capped at
  `LOG_MESSAGE_SIZE`, comfortably above the I-Bus max frame length.
- The I-Bus runs at 9600 baud, so the 115200-baud log stream can never fall
  behind the bus.

A gateway-side decoder needs one regex plus the message tables already
documented in `lib/ibus.h` (device IDs at lines 14–46, commands throughout).

### Write path — raw frame transmit

```
SEND IBUS <src> <len> <dst> <data...> <ck>
```

Parsed at `ui/cli.c:751`. Note the quirks: the `<len>` token and the final
token (`<ck>`) are accepted but **ignored** — `IBusSendCommand` recomputes
both — so the CLI accepts a full raw frame as copied from a log line.
Example (RAD → IKE device-status request):

```
SEND IBUS 68 03 80 01 EA
```

This is the full actuation surface of the firmware: anything BlueBus itself
can do (door lock jobs, LM lighting jobs, cluster text, sensor status
requests) can be triggered from the gateway by replaying the corresponding
frame. **This power is also the main hazard — see Safety below.**

## Tag Data Catalog (what the demo can show)

All of these are messages the firmware already decodes (parsers in
`lib/ibus.c`, constants in `lib/ibus.h`); the Ignition module re-implements
the same decodes in Java.

| Proposed tag path | Source message | Typical rate |
|---|---|---|
| `[BlueBus]drivetrain/speed` (km/h) | IKE 0x18 speed/RPM broadcast | ~0.5 Hz while ignition on |
| `[BlueBus]drivetrain/rpm` | IKE 0x18 | ~0.5 Hz |
| `[BlueBus]drivetrain/gear` | IKE sensor status (`IBUS_IKE_GEAR_*`) | on change |
| `[BlueBus]engine/coolantTemp` | IKE 0x19 temperature | periodic / on change |
| `[BlueBus]environment/ambientTemp` | IKE 0x19 | periodic |
| `[BlueBus]electrical/ignitionPosition` | IKE 0x11 ignition status | on change |
| `[BlueBus]body/doors/*` (4 doors, hood, trunk) | GM 0x7A doors/flaps status | on change |
| `[BlueBus]body/centralLock` | GM 0x7A | on change |
| `[BlueBus]pdc/front[1-4]`, `pdc/rear[1-4]` (cm) | PDC sensor response (8 distances, `IBUSPDCStatus_t`) | sub-second while PDC active |
| `[BlueBus]lighting/*` (blinkers, beams, parking) | LCM 0x5B light status | on change |
| `[BlueBus]vehicle/vin` | LCM redundant data / stored config | at startup |
| `[BlueBus]obc/*` (range, avg consumption…) | IKE 0x24 OBC text properties | periodic |
| `[BlueBus]_status/*` (connected, framesReceived, lastFrameAge) | module diagnostics | 1 Hz |

Ignition then gives the historian, alarming (ISA-18.2 severities on coolant
temp, door-open-while-moving, etc.), and trending for free — that is the demo
punchline: *a 2006 truck with a plant historian*.

### Demo actuations (write tags / Perspective buttons)

| Action | Mechanism | Safety class |
|---|---|---|
| Write text to the cluster / radio display ("HELLO FROM IGNITION") | IKE/RAD display write commands | Benign |
| Request sensor status (poll refresh) | IKE sensor status request | Benign |
| Flash parking lamps / angel eyes | LM diagnostic job (same one `HandlerIBusLMActivateBulbs` uses) | Demo-only, stationary |
| Lock / unlock doors | GM ZKE3 jobs (`IBusCommandGMDoor*` equivalents) | Gated, stationary only |

**Safety:** the Ignition module must ship with a **frame whitelist** — only
enumerated, reviewed frames may be transmitted, writes disabled by default,
and an interlock tag (e.g., `speed == 0`) required for anything that moves
metal. Raw passthrough of arbitrary tag writes to `SEND IBUS` must not exist
in the shipped module.

## Ignition-Side Integration Options

| Option | Effort | Fit |
|---|---|---|
| **A. Custom gateway module (`outlaw-bluebus`) with a managed tag provider** — modeled directly on `xign/outlaw-signalk` (serial client ⇄ decoder ⇄ lazy tag creation, `_status` diagnostics, native alarms) | Medium | **Recommended.** Proven pattern in the workspace; first-class tags, quality, timestamps; showcase-grade |
| B. Sidecar script (Python on the Pi) publishing MQTT → MQTT Engine | Low-medium | Works, but adds a broker + Cirrus Link modules to the demo stack; less impressive as an xign artifact |
| C. Sidecar script mapping I-Bus → **Signal K deltas** → reuse `outlaw-signalk` unchanged | Low | **Fastest possible demo** — zero new Java. Semantically odd (car as vessel) but Signal K's model is generic enough. Good weekend proof-of-concept before building A |
| D. Ignition Serial Support module + Jython tag-write scripts | Low | Fragile long-running serial handling in scripting; fine for a spike, not for the showcase |

Recommended path: **C as the proof-of-concept, A as the deliverable.** The
decode logic written for C ports directly into A.

Module design detail for option A lives in
`~/Developer/xign/docs/OUTLAW_BLUEBUS_MODULE_PROPOSAL.md`.

## Optional BlueBus Firmware Enhancements

None are blockers, but each tightens the integration (all fit the extension
points listed in [project-evaluation.md](project-evaluation.md)):

1. **Machine-readable telemetry mode** — a `SET TELEMETRY ON` that emits
   framed lines (e.g., `!IBUS:80,05,BF,18,3A,2E,5C`) without the human log
   prefix, immune to log-format drift between firmware versions.
2. **Decoded-event stream** — optionally emit already-decoded values
   (`!SPEED:116`) so the gateway needs no I-Bus knowledge; the firmware
   already decodes all of them.
3. **Bluetooth-state telemetry** — connected device, A2DP metadata, and call
   state over the CLI, so the HMI can show the audio side too.
4. **TX echo logging** — a mirror log line for frames BlueBus itself
   transmits, giving Ignition a complete bus picture (RX logging exists;
   self-TX visibility should be verified on hardware).

## Risks & Limitations

- **Log-format coupling.** Option A/C parse a debug log format that isn't a
  stability contract. Mitigate with firmware enhancement #1, or pin the
  BlueBus firmware version for the demo.
- **Data rates are automotive, not industrial.** IKE broadcasts are
  seconds-scale; this is an HMI/historian demo, not high-speed acquisition.
  Set tag group rates accordingly (~250 ms scan of the serial buffer is
  plenty).
- **Port contention.** Firmware updates and the CLI share the one serial
  port; the module needs a clean disable/release action (a `_control/enabled`
  tag).
- **Vehicle power.** A Pi-class gateway needs ignition-switched power with
  graceful shutdown (supercap/UPS HAT) — an X5 battery will not appreciate an
  always-on NUC. BlueBus itself already handles this (61 s bus-silence
  power-off).
- **Licensing.** The 2-hour Ignition trial reset is acceptable for a demo;
  Ignition Edge is the right long-term SKU for an in-vehicle gateway.
- **Do not demo write actions while driving.** Whitelist + stationary
  interlock as above; treat the I-Bus with the same respect as a live plant
  bus.

## Suggested Demo Storyboard

1. **Cold open:** tablet shows a Perspective dashboard (xign glass components
   — `outlaw-mirage` cards, `outlaw-faceplate-gauge` for speed/RPM) with the
   truck asleep; `_status` shows the bus quiet.
2. Unlock the car with the fob → door/lock tags flip live on the dashboard,
   welcome lights event visible.
3. Ignition on → speed/RPM gauges wake, coolant temp starts trending on a
   historian chart.
4. Shift to reverse → PDC bar display renders all 8 sensor distances in
   Perspective, mirroring what BlueBus writes to the cluster.
5. Press a Perspective button → **"IGNITION SCADA" appears on the instrument
   cluster** — the money shot for a controls audience.
6. Close on the alarm journal + historian trend of the drive: a 20-year-old
   vehicle with an ISA-18.2 alarm pipeline.
