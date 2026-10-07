# Vendored firmware notice

`firmware-airplay2/` is a vendored copy of
[rbouteiller/airplay-esp32](https://github.com/rbouteiller/airplay-esp32),
pinned at upstream commit `811d5f8af750096683dcec7c95d1cbc08c2f7601`
(tag `v0.2.1`, 2026-09-21).

Its nested `.git` and `.pio/` build directory were stripped before committing;
the upstream license (non-commercial use only — see
`firmware-airplay2/LICENSE`) applies unchanged.

## Local modifications vs upstream

- `config/sdkconfig.user.elegoo-wroom` — added: our board layer
  (ELEGOO ESP-WROOM-32, no PSRAM, 4 MB flash, I2S pins 26/25/22, mute GPIO 21)
- `user_platformio.ini` — added: the `elegoo-wroom` build environment
- `data/bg/` — removed: ST7789 display background, unused (no display) and
  too large for the 4 MB SPIFFS partition
- `components/u8g2/` — pruned to what the IDF build uses (`csrc/` + build
  files + LICENSE/README): `doc/`, `tools/`, `sys/`, `cppsrc/`, `pkg/` and
  `ChangeLog` deleted (~80 MB of fonts/docs). Verified both build targets pass
  after pruning.

To update: re-clone or pull upstream into a scratch dir, re-apply the three
changes above, and update the pinned commit in this file.
