# Feature Proposals: BMW E53 X5 (2006)

The E53 X5 — especially the 2004–2006 facelift ("LCI") — is one of the best-equipped
I-Bus vehicles BlueBus supports. It carries E39-generation electronics
(BlueBus detects it as `IBUS_VEHICLE_TYPE_E38_E39_E52_E53`,
`firmware/application/lib/ibus.c:1645`) and, when optioned, has a MK4 nav
computer with 16:9 board monitor (BMBT), DSP amplifier, PDC, multi-function
steering wheel, IHKA automatic climate, self-leveling rear suspension (EHC),
and optional tire-pressure monitoring (RDC).

Everything below is grounded in the existing codebase: each proposal names the
extension point it would use. Proposals are grouped by effort/risk.

Legend for the *Bus access* column:
- **Broadcast** — data already arrives on the I/K-Bus unsolicited; BlueBus only needs to parse it.
- **Poll** — data must be requested (status request or diagnostic job) on a timer.
- **Diag job** — output action performed via a diagnostic IO command to a module
  (same mechanism the existing parking-lamp / welcome-light features use).

---

## Tier 1 — Low effort, high value (firmware-only, uses existing plumbing)

### 1.1 Comfort mirror fold on lock/unlock  *(finishes an announced feature)*
`CONFIG_SETTING_COMFORT_MIRRORS` already exists in `lib/config.h:49` but has no
handler logic anywhere in the tree. The E53 with memory mirrors supports
fold/unfold via diagnostic jobs to the driver's-door mirror module. Implement
exactly like auto-lock: subscribe to the GM remote-key event
(`HandlerIBusGMRemoteKeyEntry`, `handler/handler_ibus.c:710`) and issue the
fold job on lock / unfold on unlock or ignition.

- Bus access: Diag job (mirror memory module)
- Extension points: existing config slot `0x27`, BMBT Comfort menu, `SET COMFORT` CLI
- Risk: low — same pattern as welcome lights; needs on-vehicle job verification per mirror variant

### 1.2 Steptronic gear indicator on screen
The gear-position enums are already decoded (`IBUS_IKE_GEAR_*`,
`lib/ibus.h:566-576`) and the reverse-gear path is already consumed by the
"lower volume on reverse" feature (`handler_ibus.c:1806`). Display the current
gear (P/R/N/D/1–6) on the BMBT dashboard and as a suffix on single-line UIs.

- Bus access: Broadcast (IKE sensor status 0x18/0x19)
- Extension points: BMBT dashboard (`ui/bmbt.c`), `menu_singleline.c`
- Risk: minimal — pure display

### 1.3 Extended OBC dashboard: battery voltage, fuel level, range
The BMBT dashboard already shows coolant/ambient/oil temperature. The IKE and
LCM expose more: battery voltage is available from the LM diagnostics the
firmware already polls for lighting state (`HandlerTimerIBusLightingState`,
`handler_ibus.c:2077`), and OBC values (range, average consumption, average
speed) arrive as OBC text properties (`IBUS_CMD_IKE_OBC_TEXT` 0x24). Add a
configurable second dashboard page.

- Bus access: Broadcast + Poll
- Extension points: `CONFIG_SETTING_BMBT_DASHBOARD_OBC`, BMBT UI
- Risk: low

### 1.4 Sunroof / windows comfort-close on lock
The E53's GM supports diagnostic IO jobs for window regulators and sunroof —
the same ZKE3 sub-module job mechanism the firmware already uses for
`IBusCommandGMDoorLock*` (`lib/ibus.c:2011+`). Offer "close all openings when
locking with the key fob" (mirrors the factory comfort-close held-key behavior,
but as a single-press convenience).

- Bus access: Diag job (GM ZKE3, per-variant)
- Extension points: spare config slot `0x29`, GM remote-key event
- Risk: **medium despite low code effort** — moving glass on a one-shot command
  needs care; document that factory anti-trap protection remains active, and
  gate the feature default-off. Verify jobs per GM variant (GM1 vs GM5, both
  already detected and stored in `CONFIG_GM_VARIANT`).

### 1.5 True graphical PDC on the board monitor
Visual PDC today writes text distances to the cluster/radio display
(`HandlerIBusPDCSensorUpdate`, `handler_ibus.c:1587`; all 8 sensor distances
are already parsed into `IBUSPDCStatus_t`). The MK4 GT supports static screen
writes (`IBUS_CMD_GT_*` write-static/write-zone, `lib/ibus.h:172-189`) —
enough to render a top-down bar-segment view of all four front + four rear
sensors while maneuvering, similar to the factory PDC screen on later cars.

- Bus access: Broadcast (PDC sensor response) + GT screen writes
- Extension points: existing `CONFIG_SETTING_VISUAL_PDC` (add a "graphical" mode), `ui/bmbt.c`
- Risk: low-medium — GT write bandwidth is limited; throttle updates (~2 Hz)

---

## Tier 2 — Medium effort (new polling or new Bluetooth surface)

### 2.1 SMS / message display via MAP profile
The MAP link type is already tracked in `lib/bt/bt_common.h` but has no
feature logic. BC127 supports MAP natively; BM83 support must be confirmed
per firmware. Show incoming SMS sender + preview on the BMBT (reusing the
TEL text-layout commands already implemented for caller ID,
`lib/ibus.h:405-507`), with a read-aloud-free, glance-only UX.

- Bus access: n/a (Bluetooth-side)
- Extension points: `handler_bt.c`, TEL UI paths in `ui/bmbt.c`
- Risk: medium — profile behavior varies by phone OS; iOS restricts MAP for
  non-carkit devices (works when BlueBus is paired as hands-free)

### 2.2 BLE configuration & companion app
BLE is tracked as a link type but unused. Exposing the existing config
get/set surface (everything the serial CLI in `ui/cli.c` can already do) over
a BLE GATT service would let owners configure BlueBus from a phone instead of
navigating vehicle menus — and enables future features like firmware-update
delivery and diagnostics export. The CLI command layer is already a clean
string-based API to bridge.

- Bus access: n/a
- Extension points: `lib/bt/bt_bm83.c` (BM83 BLE), `ui/cli.c` command layer
- Risk: medium — protocol design + app development; HW1/BC127 BLE differs from BM83

### 2.3 RDC tire-pressure warnings on screen (optioned vehicles)
Late E53s with RDC have the module on the bus; pressure/temperature per wheel
is retrievable via diagnostic requests. Poll while driving (reuse the
scheduled-task pattern of `HandlerTimerIBusLightingState`) and raise a
BMBT/IKE text warning on deviation — richer than the factory idiot light.

- Bus access: Poll (diag)
- Extension points: `lib/timer.c` scheduled task, IKE text write (`IBUS_CMD_IKE_WRITE_NUMERIC`/CCM text)
- Risk: medium — RDC diagnostic protocol needs on-vehicle reverse-engineering; feature must degrade gracefully when the module is absent (module-discovery bitfield `IBusModuleStatus_t` already exists for this pattern)

### 2.4 EHC (self-leveling suspension) status display — E53-specific
The E53 is the only BlueBus-supported platform with EHC rear air suspension.
Ride-height and compressor status are available via diagnostic jobs. A BMBT
status page (and a height-fault early warning) would be a genuinely
X5-exclusive feature no other integration offers.

- Bus access: Poll (diag)
- Extension points: new BMBT settings/status page, scheduled task
- Risk: medium — diag job discovery required; read-only (no height control —
  actuating suspension from an aftermarket module is out of scope for safety)

### 2.5 Seat-heating auto-on below configurable temperature
Ambient temperature is already parsed from the IKE. On cold starts, issue the
IHKA/seat-module diagnostic job to enable driver (optionally passenger) seat
heat at a configurable stage. Turn off automatically after a timer.

- Bus access: Broadcast (temp) + Diag job (seat module)
- Extension points: spare config slot, ignition-status event (`handler_ibus.c:891`)
- Risk: medium — job discovery per seat-module variant; default-off

### 2.6 AVRCP browsing (folder/playlist navigation)
Current AVRCP support covers metadata + transport control. AVRCP 1.4 browsing
would let users pick playlists/albums from the BMBT list UI (the directory
list layout used by the PBAP phonebook, `ui/bmbt.h:87-102`, is reusable).
Also: finish the empty `HandlerTimerBTBM83AVRCPPlaybackState()` stub and the
unhandled BM83 AVRCP PDUs (`lib/bt/bt_bm83.c:706`) as groundwork.

- Bus access: n/a
- Extension points: `lib/bt/bt_bm83.c`, list UI in `ui/bmbt.c`
- Risk: medium — phone-side support varies

---

## Tier 3 — Larger projects / stretch

### 3.1 Trip & vehicle data logging with export
Log speed, RPM, temperatures, fuel level, and fault events to spare EEPROM (or
stream over BLE per 2.2) for trip statistics and gentle diagnostics ("battery
voltage trending down"). The E53's IKE broadcasts speed/RPM continuously
(`IBUS_CMD_IKE_SPEED_RPM` 0x18) — the data is already flowing through
`HandlerIBusIKESpeedRPMUpdate`.

### 3.2 DSP amplifier scene control
E53s with the DSP amp already get S/PDIF audio from BlueBus. Expose DSP EQ
preset / echo & reverb scene selection from the BlueBus menu using the
existing `IBUS_CMD_DSP_SET_CONFIG` (0x36) plumbing, so audio settings live in
one place.

### 3.3 Voice-assistant deep integration
The MFL voice button already toggles Siri/Google Assistant. Extend with
press-vs-hold differentiation (constants already exist:
`IBUS_MFL_BTN_EVENT_VOICE_PRESS/HOLD/REL`, `lib/ibus.h:550-557`) — e.g., short
press = assistant, hold = redial or next paired device.

### 3.4 Check-Control message mirroring to phone (via BLE)
CCM text messages (`IBUS_CMD_IKE_CCM_WRITE_TEXT`) could be forwarded over BLE
so a companion app records "CHECK ENGINE OIL LEVEL"-style events with
timestamps — useful history for a 20-year-old vehicle.

### 3.5 Expanded phonebook
Raise the 8-contact × 3-number PBAP limit (`lib/bt/bt_common.h:97-202`) using
spare EEPROM (paired-device storage starts at `0x100`; the 1 MB-flash PIC24
also has room for a flash-backed contact store), with alphabet jump navigation
in the BMBT directory UI.

---

## Explicitly Out of Scope (safety / practicality)

- **Actuating suspension, brakes, or steering** — read-only for EHC and
  chassis systems.
- **Alarm (DWA) manipulation** beyond status display.
- **Airbag/SRS (via diagnostic bus)** — no interaction.

## Suggested Sequencing

1. **1.1 Comfort mirrors** (finishes an existing config item) → **1.2 gear
   display** → **1.3 extended dashboard** — three quick wins, all shippable in
   one release.
2. **1.5 graphical PDC** as the headline visual feature.
3. **2.2 BLE config app** as the platform investment that de-risks 2.3, 3.1,
   and 3.4.
4. Diagnostic-job features (**1.4, 2.3, 2.4, 2.5**) as on-vehicle
   reverse-engineering time permits — each should check the module-presence
   bitfield and degrade gracefully on non-optioned vehicles.
