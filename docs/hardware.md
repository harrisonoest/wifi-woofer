# Hardware & Wiring

## Parts

| Part | Notes |
|---|---|
| ELEGOO ESP32 dev board | ESP-WROOM-32, USB-C, 4 MB flash, no PSRAM |
| PCM5102 I2S DAC module | Purple/breakout board, "PCM5102 GY-PCM5102 I2S" |
| TPA3118 mono amp | Amazon B07Q6RGVHQ, analog line-in, 3 W–60 W class-D mono |
| Speaker | Matched to TPA3118 supply voltage & load |
| 5 V ≥2 A supply | ESP32 + PCM5102 (TPA3118 can share or use its own higher-voltage supply) |

## Signal path

```
ESP32                PCM5102               TPA3118
─────                ───────               ───────
GPIO (BCLK) ────────► BCK
GPIO (LRCLK)────────► LCK (WS)
GPIO (DATA) ────────► DIN
GND ────────────────► GND ────────────────► GND (signal ground)
                     LOUT ───────────────► IN  (+)
                     GND  ───────────────► IN  (–)
```

### PCM5102 module settings (solder jumpers on the back)

| Jumper | Set to | Why |
|---|---|---|
| H1L / H1R | **L** | Leave 3.3 V I2S levels, no external pull-ups needed |
| FLT | **L** (filter: normal / I2S) | Standard I2S mode |
| DEMP | **L** | No de-emphasis |
| XSMT | **H** (or wire to an ESP32 GPIO) | Un-mute; wiring to a GPIO lets firmware mute the DAC |
| FMT  | **L** | I2S format |
| SCK  | **L** | DAC generates its own system clock from BCK (44.1 kHz family) |

### TPA3118 board

- Input: the board's analog input, wire LOUT → IN+, board GND → IN–. Keep this
  wire short and shielded/twisted if possible.
- Power: per board variant, 4.5 V–24 V DC. Higher voltage = more power into the
  speaker. A 12–19 V laptop-style supply works well for most speakers.
- Gain: set by the gain resistor on the board; default 26 dB–36 dB range — start
  low and raise volume in firmware first.

## Power

- Power the ESP32 and PCM5102 from a clean 5 V rail.
- Do **not** route the amp's high-current ground through the DAC's ground path —
  star-ground at the PSU to avoid hum.
- Common ground between all three boards is required.

## Bring-up order

1. Flash the test-tone firmware, verify 1 kHz sine at the PCM5102 LOUT with a
   scope or headphones — before connecting the amp.
2. Connect the amp at low gain, speaker attached, volume 10 %.
3. Then move on to WiFi + RAOP.
