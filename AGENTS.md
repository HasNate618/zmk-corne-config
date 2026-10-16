# Agent handoff — ZMK Corne (nice!nano v2)

## Project

- **Repo:** `HasNate618/zmk-corne-config`
- **Keyboard:** Corne V3 Choc, wireless, nice!nano v2 (chip-down / standard orientation)
- **Board ID:** `nice_nano@2.0.0//zmk` (not flipped)
- **Build:** GitHub Actions workflow `Build ZMK firmware` on push to `config/**` or `build.yaml`
- **Firmware artifacts:** GitHub Actions → latest run → `firmware` artifact; local copies in `firmware/`

## Current firmware state (as of 2026-09-20)

| Half | UF2 | Role | Notes |
|------|-----|------|--------|
| Left | `corne_left-nice_nano@2.0.0__zmk-zmk.uf2` | **peripheral** | Column 0 remapped to pin **009**; P0.31 hogged; no pointing |
| Right | `corne_right-nice_nano@2.0.0__zmk-zmk.uf2` | **central** | Mouse/pointing lives here; advertises to host as "Corne" |
| Reset | `settings_reset-nice_nano@2.0.0__zmk-zmk.uf2` | — | Clears bonds/settings on both halves |

**Roles are swapped from ZMK's convention** (upstream defaults to left = central). Done to test
whether the left-half drain followed the central role — it did not. See the battery section.

Latest verified build: commit `e88989f` (all 8 CI jobs success).

### Flashing helper

```bash
./flash                                # both halves, left first, with settings_reset
./flash --side right --no-reset        # right only, keep bonds (keymap/overlay-only changes)
./flash --side both --order right-first
./flash --side left --skip-ci          # use local firmware/ files, skip GitHub
```

Waits for the latest CI run (if still building), downloads the `firmware` artifact, then polls
`lsblk` for the `NICENANO` bootloader drive per half. Default is `--reset`; use `--no-reset` for
single-half keymap/overlay fixes. Role or bond changes need `--reset` on **both** halves.

## Hardware mods (left half only)

### Dead pin P0.31 (silkscreen `031`, pro_micro 21)

- **031 GPIO is dead/shorted to GND** — likely damaged during nano/battery soldering.
- **Trace cut** between **031 pad** and **Tab column** (column still works on PCB).
- **Wire:** Tab column pad (or column side of cut) → nano pin **009** (P0.09, `pro_micro 10`).
- **Do not** jumper 031 → 009 while 031 is still on the column net (pulls column to GND).
- Pin **031** may still be soldered in header; it is unused by firmware.

### Battery

- User bridges **B+ ↔ RAW** on PCB (no physical switch).
- Either **B+ or B-** disconnect is enough to stop battery power while soldering; both + tape is safer.
- Battery sits under PCB cutout (not sandwiched under nano).

### Headers

- Mill-max / pin headers soldered on nano and PCB.
- Removing pin 031 was attempted; trace-cut + wire to 009 was the working fix.

## Config files

```
config/
  corne.keymap          # Shared keymap; 4 layers (default/lower/raise/mouse)
  corne_left.overlay    # col0 = pro_micro 10 (009) instead of 21 (031); hog P0.31
  corne_left.conf       # PERIPHERAL role, PHY_2M off, 10 min sleep, 30-50 ms interval
  corne_right.conf      # CENTRAL role, pointing, dual-battery proxy+fetching, name "Corne"
  west.yml
flash                   # flash helper (latest CI + selective side)
```

**Important:** Do **not** add `config/corne.conf` — it overrides per-half `corne_left.conf` / `corne_right.conf`.

**Important:** User uses **non-flipped** firmware only (`nice_nano@2.0.0//zmk`). Ignore `firmware/flipped/` unless they switch orientation.

## Matrix pin reference (left half, nice!nano silkscreen)

| Role | Pro Micro | nRF | Silkscreen |
|------|-----------|-----|------------|
| Row 0 | 4 | P0.22 | 022 |
| Row 1 | 5 | P0.24 | 024 |
| Row 2 | 6 | P1.00 | 100 |
| Row 3 | 7 | P0.11 | 011 |
| Col Tab/Ctrl/Shift | **10** (remapped) | P0.09 | **009** |
| Col Q/A/Z | 20 | P0.29 | 029 |
| Col W/S/X | 19 | P0.02 | 002 |
| Col E/D/C | 18 | P1.15 | 115 |
| Col R/F/V | 15 | P1.13 | 113 |
| Col T/G/B | 14 | P1.11 | 111 |

Firmware column order in overlay: `10, 20, 19, 18, 15, 14`.

## Flash procedure

Host: Linux. Bootloader appears as USB drive **`NICENANO`** (mount e.g. `/run/media/nate/NICENANO`).

```bash
# Mount if needed
udisksctl mount -b /dev/sda

# Per half: settings_reset → double-tap reset → corne_left or corne_right
cp firmware/settings_reset-nice_nano@2.0.0__zmk-zmk.uf2 /run/media/nate/NICENANO/
# wait for NICENANO to reappear
cp firmware/corne_left-nice_nano@2.0.0__zmk-zmk.uf2 /run/media/nate/NICENANO/   # or corne_right
```

**Split reset (both halves):** flash `settings_reset` on **both**, then flash each half’s firmware, then power **both on at the same time** before host BT pairing.

**NICENANO drive = bootloader, not keyboard mode.** If user says keys don’t work over USB, check they’re not stuck in bootloader.

Automated: `./flash` (see "Flashing helper" above). Manual pattern: poll `lsblk` for the `NICENANO` label, `udisksctl mount -b`, copy UF2, `sync`.

## Bluetooth pairing (known pain point)

### Symptom

Linux `bluetoothctl pair` → `org.bluez.Error.AuthenticationRejected`  
ZMK log equivalent: “Rejecting pairing request to taken profile 0” (stale bond).

### Fixes / settings actually in the firmware

- `CONFIG_BT_CTLR_PHY_2M=n` (both halves) — 2M PHY off
- `CONFIG_ZMK_BLE_PASSKEY_ENTRY=n` (both halves)
- `CONFIG_ZMK_SLEEP=y` + `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000` (10 min, both halves)
- `CONFIG_ZMK_PM_SOFT_OFF=y` (both halves)
- 30-50 ms peripheral interval (`CONFIG_BT_PERIPHERAL_PREF_MIN_INT=24`, `MAX_INT=40`, left only)
- Right central: `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y` + `PROXY`/`FETCHING` battery reporting
- Right central: `CONFIG_ZMK_KEYBOARD_NAME="Corne"` (the right shield has **no** name default)

**Not** present (older notes claimed otherwise): `CLEAR_BONDS_ON_START`, `EXPERIMENTAL_FEATURES`,
`TX_PWR+8`. TX power is stock on both halves.

**Trap:** `CONFIG_BT_PERIPHERAL=n` on the right does nothing — `BT_CLASSIC` in Zephyr `select`s
it, so Kconfig prints a warning and the value stays `y`. Harmless; the central role still works.

### Keymap bindings for Bluetooth

All on the **mouse layer** (layer 3), reached by holding `&mo 3` from **both** lower and raise
(left thumb on raise, right thumb on lower):

- **`BT_CLR_ALL`** — right half, top-right key (the `BSPC` position on the default layer)
- **`BT_SEL 0/1/2`** — right half, mouse row 2 (`M` / `,` / `.` positions): laptop / phone / tablet
- **`&soft_off`** — right half, last key of mouse row 2 (the `\` position)

### Pairing procedure

1. Forget/remove **Corne** on host (`bluetoothctl remove <MAC>`)
2. USB plug the **central** half (currently the right), mouse layer → `BT_CLR_ALL`, then unplug USB
3. Both halves on battery; wait ~10s for split pair
4. Pair from host (USB unplugged)

### Verify on host

```bash
bluetoothctl scan on
bluetoothctl devices | grep -i corne   # name is set explicitly on the right-central
```

The host-facing identity address moved to the **right** half when roles were swapped (observed
`E1:64:2E:AE:6A:44`, a random-static address that can change after `settings_reset`). Do not
hardcode it — discover by name. USB HID works when firmware is loaded (kernel log:
`ZMK Project Corne Keyboard`), but only while that half is USB-connected.

## Battery drain (left half) — UNRESOLVED

### Symptom

Left drains far faster than right. Observed: left sat at **0% for ~8 h** then died while right
still read 90%. Another run: left ~50% vs right ~96% over the same window (~8x apparent rate).

### Ruled out

| Hypothesis | Test | Result |
|---|---|---|
| Weak or mismatched LiPo packs | charge both fully, rest 30 min | both **4.13 V** — packs and charge path are healthy |
| Central role draws more | moved central to the **right** (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`) | left still drained faster → role is **not** the cause |
| `CONFIG_ZMK_POINTING` idle polling | moved from left to the right central | did not help |
| P0.31 driven high / pull-up | hogged as `input` (`p031_nc` in `corne_left.overlay`) | did not help — but this only tests RUN/IDLE, see Conclusion |

### Conclusion — cause NOT established

The P0.31 / `031` short is still the leading suspect but it has **not** been proven, and the
earlier "hog didn't help, so the nano is internally damaged" inference was invalid:

- The hog only applies in RUN/IDLE. It never tested the sleep path, so it says nothing about
  drain during deep sleep.
- Deep sleep on nRF52 is System OFF, where non-wakeup GPIOs are **disconnected**. P0.31 is not in
  the `kscan` col/row lists (column 0 is on `009`), so it is not a wakeup source — a passive pad
  short should already be harmless during deep sleep, hog or no hog.
- `4.13 V` on both halves proves equal **charge**, not equal **capacity**. A worn 500 mAh cell
  still charges to 4.13 V but delivers noticeably less.

So the extra left drain could be the P0.31 short, a worn left cell, or another board-level leak.
Use the test below to separate them.

### Test to settle it (no soldering)

1. Charge both halves to full, rest 30 min, record rested voltage + `corne-battery` for each
2. Leave **both halves completely untouched for 12 h** (both must sit in deep sleep, not idle)
3. Record both again

- Left drops clearly more while both were asleep → **not** P0.31 (it is disconnected in System
  OFF) → suspect left cell capacity or a board-level leak
- Both hold → deep sleep is doing its job; the earlier "8x" was mostly active/idle time plus the
  flat-bottom gauge

To isolate capacity specifically, repeat the same 12 h run once per half with the other half off.

### Options

1. Inspect/wick the P0.31 pad under magnification for a solder bridge — no-risk, try first
2. Lift/cut nano header pin 21 (`031`) — **risky**, sits next to the `009` repair
3. Replace the nano (~$12) — definitive
4. Live with it + a physical power switch — no board risk

### Mitigation

Physical power switch: remove the `B+ ↔ RAW` bridge on the PCB and solder a slide switch across
those same two pads. **Do not cut the nano pin.** Disconnect the battery first (`B-` then `B+`,
tape both). Do the left half at minimum; both halves ideally so the packs age together.

A short that survives System OFF would still draw during sleep and `soft_off`, so the switch is
the only guaranteed fix. If the 12 h test shows that deep sleep holds voltage, then nightly
`&soft_off` becomes an alternative — but note ZMK has **no auto soft-off timeout**, `&soft_off`
is manual (mouse layer, `\` key), and a switch is the only thing that helps if the MCU hangs or
is left in the bootloader.

### Gauge caveat (expected behavior, not a bug)

ZMK's `lithium_ion_mv_to_pct()` is a **linear** map: 4200 mV = 100%, 3450 mV = 0%. A real LiPo
is flat through the mid-range and steep at the bottom, and pack protection cuts at ~2.8-3.0 V.
So "0%" can display for many hours of real runtime. Only compare **rested** voltages (30 min
off the charger); a central half also sags more under load than a peripheral.

### Measuring

Multimeter on 20 V DC. Black on `B-`/GND, red on `B+`/VBAT pads. Full and rested should read
~4.15-4.20 V on both halves. **Never** probe in ammeter mode across `B+`/`B-` — that shorts the
LiPo. Idle-current measurement requires breaking the `B+` wire to put the meter in series.

### Host-side readout

`~/.local/bin/corne-battery` prints `{"left":N,"right":N,"central":N,"peripheral":N,"swapped":true}`.
It auto-discovers the device by name (no hardcoded MAC) and labels by the `0x2901` descriptor:
the characteristic **with** the descriptor is the peripheral, the one without is the central.
After the role swap `central` = physical right, `peripheral` = physical left.

## Build / commit notes

- Push to `master` triggers CI.
- `config/corne_left.overlay` must **not** `#include "corne.dtsi"` (breaks user-config build).
- `config/corne.keymap` needs `#include <dt-bindings/zmk/bt.h>` for `BT_CLR_ALL`.
- User prefers **not** committing unless asked; `firmware/flipped/` is untracked local artifact.

## Testing keys without switches

Short nano pins with tweezers (USB in, text editor open):

- **009 + 022** → Tab (left column 0)
- **029 + 024** → A
- Touch both switch pads at a key to test diode/switch joint.

## Keymap layout (4 layers)

- **default** — QWERTY; `&mo 1` (lower, left thumb), `&mo 2` (raise, right thumb)
- **lower** — numbers/symbols; right thumb is `&mo 3` (mouse)
- **raise** — F-keys, nav (`HOME`/`PG_DN`/`PG_UP`/`END`/`INS`/`DEL`), arrows, media/brightness
- **mouse** — hold **both** lower and raise to reach it (`&mo 3` on both thumb clusters)
  - right hand: left/right click, scroll, mouse move, `BT_CLR_ALL`, `BT_SEL 0-2`, `&soft_off`
  - left hand: `MB3`/`MB4`/`MB5`
- Mod-taps: `&mt LCTRL ESC`, `&mt LSHIFT F13`, `&mt RALT F14`
- Macros: `emdash`, `rightarrow` (`wait-ms = <5>`)

`config/corne.keymap` needs `#include <dt-bindings/zmk/bt.h>` for `BT_CLR_ALL`.

## Open issues

- [ ] **Left-half battery drain** — cause not established; run the 12 h deep-sleep test first (see battery section)
- [ ] P0.31 short: decide between wicking the pad, lifting pin 21, or replacing the nano — only after the 12 h test implicates it
- [ ] Right-central is a workaround for ZMK's convention; revert only if you need upstream-default behavior
- [ ] `flash` helper is new — not yet exercised for `--order right-first` or `--skip-ci`

## Do not

- Reflash **flipped** firmware (user confirmed chip-down / non-flipped)
- Remap column 0 back to pin 031 without fixing/replacing the nano
- **Cut or desolder nano header pin 21 (`031`)** — risky, adjacent to the `009` repair; use a power switch instead
- Expect `CONFIG_BT_PERIPHERAL=n` to stick on the right half (`BT_CLASSIC` forces it back to `y`)
- Commit unless the user asks
- Assume `NICENANO` visible = keyboard broken (it's bootloader mode)
