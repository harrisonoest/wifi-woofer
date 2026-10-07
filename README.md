# esp32-speaker

An internet-connected, WiFi speaker built on an ESP32. Appears natively as an
**AirPlay 2** speaker on iPhone/iPad/Mac, with multi-room sync, Bluetooth A2DP
input, and a built-in web config UI.

## Firmware: airplay-esp32

Instead of building an AirPlay 1 (RAOP) receiver from scratch, this project uses
**[airplay-esp32](https://github.com/rbouteiller/airplay-esp32)** (forked/cloned
at `firmware-airplay2/`, pinned by commit) as the firmware base:

- True AirPlay 2 (Control Center, multi-room PTP sync, ALAC + AAC)
- Built on Shairport Sync / openairplay airplay2-receiver lineage
- Native HomeKit (HAP) implementation in firmware — Apple Home pairing without
  Homebridge; Home Assistant discovers it via AirPlay as well
- ESP-IDF v5.5 under the hood, built with PlatformIO
- **License: non-commercial use only** (fine for this project)

## Hardware

| Part | Role |
|---|---|
| ELEGOO ESP32 dev board (ESP-WROOM-32, USB-C, 4 MB flash, no PSRAM) | WiFi + AirPlay 2 decode |
| PCM5102 I2S DAC module | I2S → analog line-out |
| TPA3118 mono amp board | Analog in → speaker (analog-input class-D) |
| 5 V PSU (≥2 A) | ESP32 + DAC (TPA3118 may want its own 12–24 V supply) |

Signal path: `ESP32 → I2S → PCM5102 → analog → TPA3118 → speaker`

See [docs/hardware.md](docs/hardware.md) for wiring (still accurate) and
[docs/architecture.md](docs/architecture.md) for the design rationale.

## Build & flash

Requires [PlatformIO](https://platformio.org/) (`pip install platformio`).
This repo adds a custom board environment on top of the upstream ones:

```sh
cd firmware-airplay2
pio run -e elegoo-wroom -t upload      # firmware
pio run -e elegoo-wroom -t uploadfs    # web UI (required)
pio run -e elegoo-wroom -t monitor     # serial console
```

Our board layer (gitignore-friendly upstream convention):

- `firmware-airplay2/config/sdkconfig.user.elegoo-wroom` — PSRAM off, 4 MB
  partitions, I2S pins BCK=26 / WS=25 / DO=22, PCM5102 XSMT (mute) on GPIO21
- `firmware-airplay2/user_platformio.ini` — the `elegoo-wroom` env

## Status

- [x] Board confirmed: ESP-WROOM-32 (classic ESP32, no PSRAM)
- [x] Firmware base selected: airplay-esp32 (AirPlay 2)
- [x] Board environment configured (`elegoo-wroom`)
- [x] First build passes on this machine: firmware `SUCCESS` (RAM 24.9%, flash 90.3% of the OTA app slot), SPIFFS web UI `SUCCESS`
- [x] `data/bg/` (ST7789 display background) removed from our clone — no display, and it didn't fit the 4 MB SPIFFS partition. Re-add nothing when syncing upstream unless you add a display.
- [ ] PCM5102 arrives → wire, flash, first audio test
- [ ] Home Assistant discovery + Apple Home (HAP or Homebridge) verified
