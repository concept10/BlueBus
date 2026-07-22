# BlueBus Project Evaluation

*Firmware version evaluated: 1.4.35 (`firmware/application/mappings.h`). Evaluated 2026-07.*

## 1. What the Project Is

BlueBus is a hardware + firmware product that sits on the BMW I/K-Bus and
emulates a factory CD changer (I-Bus device `0x18`). On top of that emulation it
layers Bluetooth audio streaming, hands-free telephony with factory microphone
support, phonebook access, and a set of body/comfort features driven through
diagnostic jobs on the light module (LM) and central body module (GM).

Two hardware generations are supported at runtime from a single firmware image:

| Board | Bluetooth module | Audio path |
|-------|------------------|------------|
| HW v1 | BC127 (ASCII UART protocol) | WM8804 S/PDIF transceiver + PCM5122 DAC |
| HW v2 | BM83 (Microchip binary protocol) | DIT4096 S/PDIF encoder + PCM5122 DAC |

Board detection is automatic (`UtilsGetBoardVersion()`), and the entire
Bluetooth stack is abstracted behind `lib/bt.c`, which dispatches each
`BTCommand*` call to the BC127 or BM83 implementation.

## 2. Architecture Assessment

### Strengths

- **Clean superloop + event architecture.** `main.c` runs four cooperative
  pumps (`BTProcess`, `IBusProcess`, `TimerProcessScheduledTasks`,
  `CLIProcess`). Protocol drivers decode frames and publish numbered events
  through a small pub/sub bus (`lib/event.c`); handlers and UIs subscribe.
  This keeps protocol parsing, business logic, and presentation cleanly
  separated and makes new features low-risk to add.
- **Strong hardware abstraction.** Two entirely different Bluetooth modules
  (ASCII vs. binary protocols) live behind one API. The same discipline shows
  in the audio chain (PCM5122 always, WM8804/DIT4096 per board).
- **Auto-detection over configuration.** Vehicle platform, GM (body module)
  variant, LM (light module) variant, nav generation, and head-unit type are
  all detected from live bus traffic and persisted to EEPROM. The user rarely
  has to configure anything about their car.
- **Deep I-Bus coverage.** `lib/ibus.h` defines ~30 device addresses and
  message constants for IKE, GM (ZKE3 GM1/GM4/GM5/GM6, ZKE4, ZKE5), six light
  module variants (LME38, LCM, LCM_A, LCM_II/III/IV, LSZ, LSZ_2), four nav
  generations (MK1–MK4), PDC, DSP, MID, BMBT, IRIS, and the OE telephone UI.
  This is one of the most complete open I-Bus implementations available.
- **Robust failure handling.** All CPU trap ISRs log to EEPROM counters and
  reset; BC127 boot failures are counted with LED feedback; a power-off timer
  disables the transceiver after 61 s of bus silence to protect the battery.
- **Serviceability.** USB bootloader, serial CLI (`ui/cli.c`) with full
  get/set access to configuration, per-source debug log toggles, and a
  versioned EEPROM migration system (`upgrade.c`).
- **Localization.** 13 languages in `lib/locale.c`, used across all UIs.

### Weaknesses / Risks

- **`handler_ibus.c` and `ui/bmbt.c` are very large** (the BMBT UI alone is
  ~4,200 lines). Feature logic, per-platform branches, and timers are
  interleaved; extracting per-feature modules (e.g., `features/comfort_lock.c`)
  would improve testability.
- **No automated tests.** All validation is on-vehicle or via CLI. The event
  bus and pure parsers (I-Bus frame decode, vCard parsing, BCD phone numbers)
  are prime candidates for host-side unit tests.
- **Platform detection is coarse.** E83 (X3) and E85/E86 (Z4) have no distinct
  enum — they resolve into the `E46`/`E8X` buckets by cluster nibble. This
  works today but limits per-model tuning.
- **Known stubs and TODOs:**
  - `CONFIG_SETTING_COMFORT_MIRRORS` (`lib/config.h:49`) is defined but has
    no handler logic anywhere — an announced-but-unimplemented feature.
  - `HandlerTimerBTBM83AVRCPPlaybackState()` (`handler/handler_bt.c`) is an
    empty stub.
  - Unhandled AVRCP PDUs on BM83 log `AVRCP Not Implemented`
    (`lib/bt/bt_bm83.c:706`).
  - `@TODO` markers in `lib/ibus.c:423` (refactor), `lib/ibus.c:2301`
    (GT title length hardcoding), `bootloader/lib/protocol.c:338`.
- **BLE and MAP link types are tracked but unused** (`lib/bt/bt_common.h`) —
  wasted potential (see the E53 feature proposals: BLE configuration app,
  SMS display via MAP).
- **Max 8 PBAP contacts × 3 numbers** — a hard limit that shows its age
  against modern phonebooks.

## 3. Extension Points (for anyone adding features)

These are the seams new features should use — they exist today and require no
refactoring:

| Extension point | Location | Notes |
|-----------------|----------|-------|
| Spare EEPROM config slots | `CONFIG_SETTING_COMFORT_OPEN_SLOT0` (`0x29`), `CONFIG_SETTING_OPEN_SLOT_1` (`0x43`) in `lib/config.h` | Reserved addresses for new toggles |
| Event bus | `EventRegisterCallback()` in `lib/event.c` | Subscribe to any of ~50 `IBUS_EVENT_*` / `BT_EVENT_*` IDs |
| Scheduled tasks | `TimerRegisterScheduledTask()` in `lib/timer.c` | Periodic polling (e.g., diagnostic sensor reads) |
| LM diagnostics | `HandlerIBusLMActivateBulbs()` (`handler/handler_ibus.c:292`) | Central point for all comfort-lighting output |
| GM diagnostics | `IBusCommandGMDoor*` senders (`lib/ibus.c:2011+`) | Per-ZKE-variant body jobs; pattern for new GM IO jobs |
| BMBT settings tree | Menu indices in `ui/bmbt.h:27-85` | Add items to the Comfort/Audio/Calling pages |
| Single-line menus | `ui/menu/menu_singleline.c` | Shared engine for CD53/MID/MIR/IRIS |
| Serial CLI | `SET COMFORT` handlers in `ui/cli.c:791+` | Every setting should be CLI-settable |
| Platform detection | `IBusGetVehicleType()` (`lib/ibus.c:1645`), enums at `lib/ibus.h:561-564` | Where a new platform enum would be added |

## 4. Overall Verdict

The codebase is mature, disciplined embedded C with an unusually complete
reverse-engineered I-Bus layer and a genuinely good abstraction boundary
between the two Bluetooth stacks. Its main structural debt is concentration of
logic in two very large files and the absence of host-side tests. It is in
excellent shape to absorb the feature proposals in
[e53-x5-feature-proposals.md](e53-x5-feature-proposals.md); the E93 question is
a different matter entirely and is treated honestly in
[e93-platform-feasibility.md](e93-platform-feasibility.md).
