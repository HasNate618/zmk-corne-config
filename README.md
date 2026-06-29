# ZMK Corne — stock defaults

Stock ZMK firmware for Corne V3 Choc + nice!nano v2 / nRF52840 Pro Micro clone.

- Board (chip-down): `nice_nano@2.0.0//zmk`
- Board (chip-up / flipped): `nrfmicro@1.1.0/nrf52840/flipped_zmk`
- Shield: `corne_left` / `corne_right`
- Keymap: upstream ZMK default (no custom keymap file)

## Flash order

1. Double-tap reset → `NICENANO` drive appears
2. Copy `settings_reset-...flipped_zmk.uf2` (chip-up) or `settings_reset-nice_nano...uf2` (chip-down)
3. Double-tap reset again
4. Copy `corne_left-...flipped_zmk.uf2` (left) or `corne_right-...` (right)

Download `.uf2` files from GitHub Actions → latest workflow run → **firmware** artifact.
