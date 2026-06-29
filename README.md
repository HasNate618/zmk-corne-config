# ZMK Corne — stock defaults

Stock ZMK firmware for Corne V3 Choc + nice!nano v2 / nRF52840 Pro Micro clone.

- Board: `nice_nano@2.0.0//zmk`
- Shield: `corne_left` / `corne_right`
- Keymap: upstream ZMK default (no custom keymap file)

## Flash order

1. Double-tap reset → `NICENANO` drive appears
2. Copy `settings_reset-nice_nano-zmk.uf2` to the drive
3. Double-tap reset again
4. Copy `corne_left-nice_nano-zmk.uf2` (left) or `corne_right-nice_nano-zmk.uf2` (right)

Download `.uf2` files from GitHub Actions → latest workflow run → **firmware** artifact.
