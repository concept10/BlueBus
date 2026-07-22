# Feasibility Analysis: BMW E93 (E9x Platform)

## Summary — the honest answer up front

The E93 (3-Series convertible, 2007–2013) is part of the E90/E91/E92/E93
generation, which **does not have an I-Bus or K-Bus**. BMW replaced the
I/K-Bus architecture with **K-CAN** (body/comfort), **PT-CAN** (powertrain),
and a **MOST fiber-optic ring** (audio/infotainment). Current BlueBus hardware
is built around the Melexis TH3122 I-Bus transceiver and a CD-changer
emulation protocol that simply does not exist on this bus architecture.

**Conclusion: E93 support is not a firmware feature — it is a new hardware
variant and a substantially new firmware protocol layer.** It is achievable,
but it should be scoped as "BlueBus CAN Edition," not as an addition to the
current product. This document lays out what that would take so the decision
can be made deliberately.

> Note: do not confuse the E93 with the `IBUS_VEHICLE_TYPE_E8X` enum in the
> firmware (`lib/ibus.h:563`) — that bucket covers the K-Bus-equipped
> E83 X3 / E85–E86 Z4, not the E9x generation.

## Why the current architecture cannot carry over

| BlueBus assumption today | E9x reality |
|---|---|
| Single-wire 9600-baud I/K-Bus, TH3122 transceiver | K-CAN: 100 kbit/s fault-tolerant two-wire CAN; PT-CAN: 500 kbit/s high-speed CAN |
| Audio enters the system by emulating a CD changer with analog/S-PDIF output into the radio/DSP amp | Audio sources on nav/HiFi-professional cars live on the **MOST fiber ring**; base cars use head-unit-internal sources |
| Text/menus drawn via GT/MID/BMBT I-Bus write commands | Displays driven by the head unit (CCC/CIC) over MOST/internal buses |
| Comfort features via LM/GM diagnostic jobs on the same bus | Body functions on K-CAN via FRM (footwell module), CAS (car access), JBE (junction box) |

## What *is* feasible on an E9x

### Path A — Audio via the factory AUX/USB port + CAN for control (realistic)

Every E9x has a factory AUX input (and most have the MULF/USB module or can
take one). A "BlueBus CAN" device could:

1. **Inject audio through the AUX input** (analog; the existing PCM5122 DAC
   output stage carries over directly).
2. **Attach to K-CAN** (available at the head unit / junction box) with a
   fault-tolerant CAN transceiver (e.g., NXP TJA1055) to:
   - Read steering-wheel button presses (track next/prev, volume, voice) —
     giving real AVRCP control, which a dumb AUX cable cannot do.
   - Read ignition/terminal status, VIN, speed, reverse gear — enabling
     the same auto-power, auto-lock-style, and reverse-volume features.
   - Write radio display text on non-nav head units (the "radio text" CAN
     messages used by existing E9x display projects) for song metadata.
3. Reuse essentially all of the Bluetooth side unchanged: `lib/bt/` (BM83),
   PBAP, HFP, AVRCP, and the handler/event architecture are bus-agnostic.

**Limitations of Path A:** hands-free audio would route through the AUX input
(no factory microphone integration unless the car has the factory Bluetooth
MULF, which then competes for the phone), and on CCC/CIC nav cars the display
integration is limited because those screens are MOST-driven.

### Path B — MOST ring participation (not recommended)

Emulating a MOST audio source (like a factory CD changer / MULF on the ring)
would give first-class audio and display integration on nav cars, but MOST150/
MOST25 transceivers, ring timing, and licensing make this a major undertaking
with poor open-source precedent. Cost and complexity are out of proportion to
the market; aftermarket MOST retrofits exist commercially already.

### Path C — Comfort-only module (niche)

A K-CAN-only device (no audio) providing comfort features — comfort blinker
tuning, welcome lights, mirror fold on lock, etc. Many of these already exist
on E9x as coding options (the E9x FRM/CAS are far more codable than E39-era
modules), which undercuts the value proposition. Not recommended as a product.

## What a "BlueBus CAN Edition" would require

### Hardware
- Replace TH3122 with a fault-tolerant low-speed CAN transceiver (TJA1055 or
  similar) — K-CAN is 100 kbit/s fault-tolerant.
- The PIC24FJ1024GA606 has ECAN peripherals-class siblings; the current MCU
  **does not have a CAN controller**, so either an external SPI CAN controller
  (MCP2515/MCP2518FD) or an MCU change (dsPIC33/PIC24 with ECAN, or a
  Cortex-M with FD-CAN) is required. An MCU change is the cleaner path.
- Audio output stage (PCM5122 line-level into AUX) carries over; S/PDIF
  (DIT4096) is not useful on E9x and could be dropped from this variant.
- New harness: quadlock/AUX/K-CAN tap instead of the round-pin CD-changer
  connector.

### Firmware
- New `lib/kcan.c` peripheral driver + message layer parallel to `lib/ibus.c`,
  publishing the **same event IDs** where semantics overlap (ignition, speed,
  reverse, MFL buttons) so `handler/` logic and the Bluetooth stack are reused.
- The event-bus architecture (`lib/event.c`) is the key asset here: because
  handlers subscribe to abstract events rather than parsing frames, a large
  fraction of `handler_bt.c` and the comfort logic shape can be reused.
- No BMBT/MID/CD53 UI on E9x — a new minimal "radio text + steering wheel"
  UI module, plus the BLE configuration app (proposed as feature 2.2 in the
  [E53 document](e53-x5-feature-proposals.md)) becomes near-essential, since
  there are no vehicle menus to configure through.
- K-CAN message database: substantial reverse-engineering effort, though the
  E9x K-CAN is among the best-documented BMW buses in the open community.

### Rough effort assessment

| Workstream | Scale |
|---|---|
| Hardware variant (MCU + CAN transceiver + harness) | New board spin, prototype + validation on-vehicle |
| K-CAN driver + message layer | Weeks–months incl. reverse-engineering validation |
| Reused Bluetooth/audio/handler code | High reuse (the major cost saving) |
| New configuration surface (BLE app) | Shared investment with the E53 roadmap |

## Recommendation

1. **Do not attempt E93 support in the current firmware/hardware** — there is
   no I-Bus to speak on. Update the README's supported-vehicle table if E9x
   inquiries are common, to save support burden.
2. If E9x demand justifies it, scope **Path A as a separate hardware variant**
   ("BlueBus CAN"), explicitly reusing `lib/bt/`, `lib/event.c`, the audio
   output stage, and the handler architecture.
3. **Build the BLE configuration app first on the E53/I-Bus product** (feature
   2.2 in the E53 proposals) — it pays off immediately on current hardware and
   is a prerequisite for a menu-less E9x product.
