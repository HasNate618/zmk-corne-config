# Agent handoff — ZMK Corne (nice!nano v2)

## Project

- **Repo:** `HasNate618/zmk-corne-config`
- **Keyboard:** Corne V3 Choc, wireless, nice!nano v2 (chip-down / standard orientation)
- **Board ID:** `nice_nano@2.0.0//zmk` (not flipped)
- **Build:** GitHub Actions workflow `Build ZMK firmware` on push to `config/**` or `build.yaml`
- **Firmware artifacts:** GitHub Actions → latest run → `firmware` artifact; local copies in `firmware/`

## Current firmware state (as of 2026-07-21)

| Half | UF2 | Notes |
|------|-----|--------|
| Left | `corne_left-nice_nano@2.0.0__zmk-zmk.uf2` | Column 0 remapped to pin **009** (see hardware) |
| Right | `corne_right-nice_nano@2.0.0__zmk-zmk.uf2` | Stock |
| Reset | `settings_reset-nice_nano@2.0.0__zmk-zmk.uf2` | Clears bonds/settings on both halves |

Latest successful build: commit `e3c3e35` (includes `corne.keymap` + BLE fixes).

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
  corne.keymap          # Shared keymap; Lower+Tab = BT_CLR_ALL
  corne_left.overlay    # col0 = pro_micro 10 (009) instead of 21 (031)
  corne_left.conf       # BLE: PHY_2M off, TX_PWR+8, CLEAR_BONDS_ON_START, EXPERIMENTAL
  corne_right.conf      # BLE: PHY_2M off, TX_PWR+8, EXPERIMENTAL
  west.yml
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

Automated flash pattern used in this project: poll `lsblk` for `NICENANO` label, copy UF2, `sync`.

## Bluetooth pairing (known pain point)

### Symptom

Linux `bluetoothctl pair` → `org.bluez.Error.AuthenticationRejected`  
ZMK log equivalent: “Rejecting pairing request to taken profile 0” (stale bond).

### Fixes applied in firmware

- `CONFIG_ZMK_BLE_CLEAR_BONDS_ON_START=y` (left only)
- `CONFIG_ZMK_BLE_EXPERIMENTAL_FEATURES=y`
- `CONFIG_BT_CTLR_PHY_2M=n`
- Keymap: **Hold Lower + tap Tab** → `BT_CLR_ALL`

### Pairing procedure

1. Forget/remove **Corne** on host (`bluetoothctl remove <MAC>`)
2. USB plug left → **Lower + Tab** (BT_CLR_ALL) → unplug USB
3. Both halves on battery; wait ~10s for split pair
4. Pair from host (USB unplugged)

### Verify on host

```bash
bluetoothctl scan on
# look for "Corne"
bluetoothctl pair F0:9A:62:B9:46:2E   # MAC may change after reset
```

USB HID works when firmware is loaded (kernel log: `ZMK Project Corne Keyboard`).

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

## Open issues (last session)

- [ ] **BLE pairing** may still fail until user runs **Lower+Tab** (BT_CLR_ALL) with USB, then pairs with USB out
- [ ] Right half may need reflash if split bond drifts after one-sided settings_reset
- [ ] Consider removing `CONFIG_ZMK_BLE_CLEAR_BONDS_ON_START` after stable pairing (clears bonds every boot)

## Do not

- Reflash **flipped** firmware (user confirmed chip-down / non-flipped)
- Remap column 0 back to pin 031 without fixing/replacing nano
- Commit unless user asks
- Assume `NICENANO` visible = keyboard broken (it’s bootloader mode)
