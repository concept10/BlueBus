# BlueBus Documentation

This directory contains project evaluation and forward-looking feature documentation
for the BlueBus firmware and hardware.

## Contents

| Document | Description |
|----------|-------------|
| [project-evaluation.md](project-evaluation.md) | Architecture and code-health evaluation of the current firmware (v1.4.35) |
| [e53-x5-feature-proposals.md](e53-x5-feature-proposals.md) | Proposed additional features for the BMW E53 X5 (with emphasis on the 2006 facelift model), grounded in the existing codebase |
| [e93-platform-feasibility.md](e93-platform-feasibility.md) | Feasibility analysis for supporting the BMW E93 (E9x-generation) platform |
| [ignition-scada-evaluation.md](ignition-scada-evaluation.md) | Evaluation of BlueBus as a field device for Inductive Automation Ignition SCADA/HMI (in-vehicle demo); companion module proposal in `~/Developer/xign/docs/OUTLAW_BLUEBUS_MODULE_PROPOSAL.md` |

## Quick Orientation

BlueBus emulates a CD changer (device `0x18`) on the BMW I/K-Bus, providing
Bluetooth audio (A2DP/AVRCP), hands-free telephony (HFP/PBAP), and a growing set
of comfort features (auto-lock, comfort blinkers, welcome/follow-me lights,
parking lamps, visual PDC, reverse volume reduction).

Firmware layout (see [project-evaluation.md](project-evaluation.md) for detail):

```
firmware/application/
├── main.c                  # Boot + superloop (BT, IBus, timers, CLI)
├── handler/                # Business logic bridging I-Bus and Bluetooth
│   ├── handler_ibus.c      # Comfort features, CDC emulation, module discovery
│   ├── handler_bt.c        # Call handling, device management, BM83/BC127 boot
│   └── handler_common.c    # Shared context, volume/TEL helpers
├── lib/
│   ├── ibus.c/.h           # I-Bus driver: parsers + ~120 command senders
│   ├── bt/                 # BC127 (HW1) and BM83 (HW2) stacks behind bt.c
│   ├── config.c/.h         # EEPROM-backed settings
│   └── event.c, timer.c    # Pub/sub event bus + cooperative scheduler
└── ui/                     # CD53/MIR/IRIS, MID, BMBT (nav) UIs + serial CLI
```

Vehicle platform is auto-detected from the instrument cluster's vehicle-config
response (`IBusGetVehicleType()`, `firmware/application/lib/ibus.c:1645`). The
E53 X5 resolves to the `IBUS_VEHICLE_TYPE_E38_E39_E52_E53` family and is a
first-class supported platform today.
