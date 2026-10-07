# RAOP (AirPlay 1) Receiver — Integration Plan

Concrete strategy for bringing an AirPlay 1 receiver onto this ESP32
(ESP-WROOM-32, 4 MB flash, **no PSRAM**) by vendoring proven open-source code
rather than writing the protocol from scratch.

> ⚠️ **Honesty note**: this document was written from model knowledge of the
> upstream repos (file lists, license texts, structures). Items marked
> **[VERIFY]** must be checked against the actual repositories before/while
> vendoring. Web access was unavailable at authoring time.

## Candidate upstream projects

### 1. philippe44/RAOP-Player (primary candidate)

- Portable C RAOP/AirPlay-1 receiver, designed to be embedded (it originated as
  the RAOP capability inside philippe44's AirConnect bridge).
- Relevant modules:
  - `raop/` — RTSP session setup, RTP audio/control/timing channels, crypto
    (RSA/AES), audio reassembly. **[VERIFY]** exact file list at HEAD: at
    minimum `raop.c`, `raop.h`, `raop_utils.c/h`, `raop_rtp.c/h`,
    `raop_handlers.c/h`, `stream.c/h`, plus embedded support libs
    (`cross-openssl`, `alac`, `openssl` choice of backend).
  - `alac/` — ALAC codec. **[VERIFY]**: the canonical copy is David
    Hammerton's original ALAC decoder (`alac.c`, `alac.h`, `decomp.h`), same
    lineage as used in shairport / shairport-sync / squeezelite.
- Provenance / licensing — all obligations below apply to whatever we vendor:
  - **RAOP protocol code**: LGPL (philippe44). Obligation: dynamic-style
    linkage is the clean path; on ESP-IDF everything statically links, so we
    must (a) publish our modified versions of the LGPL sources, (b) keep them
    cleanly separable in `firmware/components/raop/`, and (c) state in the
    repo README that LGPL code is included and where. Practically every
    ESP32 project with this code does exactly this. **[VERIFY]** that
    RAOP-Player is LGPL and not GPL — some philippe44 components are GPL.
  - **ALAC decoder (David Hammerton /ALAC)**: originally released under LGPL
    ("ALAC may be used under the terms of the GNU LGPL"); Apple later released
    ALAC as Apache-2.0. Keep the original copyright header whatever copy is
    taken. **[VERIFY]** which variant RAOP-Player's `alac/` dir carries.
  - **Crypto**: RAOP-Player can use either its embedded OpenSSL port or
    mbedTLS. **Prefer mbedTLS** — it's already in ESP-IDF (`esp_mbedtls`),
    saving ~100+ KB flash. There will be small glue (RSA key setup, AES-CBC
    context) to map. **[VERIFY]** what `RAOP_SSL`/`USE_MBEDTLS`-style
    switches exist; otherwise write the glue ourselves.
- Why primary: it is the same code that squeezelite-esp32 runs on **this
  exact chip** (classic ESP32, no PSRAM), so RAM/flash budgets are known-good
  in practice.

### 2. sle118/squeezelite-esp32 — RAOP receiver component (reference + fallback)

- Contains a heavily adapted copy of the same RAOP code, already fixed for
  ESP32/lwIP/FreeRTOS. Its `components/raop/` (and the `raop_sink`-style API it
  exposes) demonstrates every shim we need: lwIP socket quirks, task
  priorities, heap-caps choices, mDNS TXT keys that actually satisfy Home
  Assistant, and audio sink callbacks. **[VERIFY]** current component path in
  that repo (it has moved between `components/raop/` and `esp32/` over time).
- License: inherits the GPL-2-or-later of the squeezelite family (and the
  underlying RAOP/Shairport GPL lineage — the receiver code descends from
  **Shairport by James Laird (GPL/MIT mix)** via forks). If we copy *from*
  squeezelite-esp32 instead of RAOP-Player, the copied files are GPL and the
  whole firmware effectively becomes GPL-distributed — acceptable for a
  personal project, but prefer LGPL RAOP-Player sources where both exist.
- Even if we vendor from RAOP-Player, keep squeezelite-esp32 open as the
  reference for every ESP32-specific adaptation point.

## Platform shim required

Upstream code is POSIX. Mappings needed:

| POSIX | ESP-IDF |
|---|---|
| BSD sockets | lwIP (`lwip/sockets.h`). Same API surface; watch: `SO_REUSEADDR`, socket buffer sizes (`SO_RCVBUF` is limited), no `fork`/`select` on many fds — upstream uses one thread per channel, we keep that (3 sockets is fine with lwIP `select`). |
| pthreads | FreeRTOS tasks: one each for the RTP data, control, and timing listeners + one decode/output task. Use `xTaskCreatePinnedToCore(…, 1)` (PRO core, low prio above IDLE). |
| mutex/cond | `SemaphoreHandle_t` / `xSemaphore`, `EventGroupHandle_t` or task notifications. |
| malloc pools | `heap_caps_malloc(n, MALLOC_CAP_8BIT)` (internal RAM only — no SPIRAM). |
| stdout/audio write | our `audio_write(const int16_t*, size_t)` from `firmware/main/audio.h`. |
| crypto | mbedTLS (ESP-IDF built-in) replacing the OpenSSL dependency. |
| clock | `esp_timer_get_time()` (µs) for NTP-era math; `gettimeofday` also works via lwIP/newlib. |

## Buffer & memory budget (no PSRAM)

- Total free internal RAM after WiFi+lwIP comes up on this target: roughly
  **~130–200 KB** **[VERIFY]** with the final IDF version — measure with
  `heap_caps_get_free_size(MALLOC_CAP_8BIT)` before locking numbers.
- Budget targets:
  - RTP reorder/jitter buffer: **64–128 KB** (p->frame fifo; ~200–400 packets
    of 352-byte ALAC frames at 44.1 kHz stereo). Start at 64 KB, tune by
    measuring underruns vs. added latency on the real network.
  - ALAC decode scratch: ~8 KB.
  - RTSP/crypto working memory: ~4–8 KB.
  - Task stacks: 3× 4 KB (socket loops) + 1× 6 KB (decode/out). **[VERIFY]**
    high-water marks with `uxTaskGetStackHighWaterMark` and trim.
- **Flash**: RAOP+ALAC+mbedTLS glue ≈ **80–150 KB** of code on top of a WiFi
  app. With 4 MB flash there is no pressure as long as the partition table
  gives the app ≥ 1.5 MB.
- Latency/quality trade-off: RAOP senders buffer 1–2 s ahead; a 64–128 KB
  internal jitter buffer is well within that.

## Known pitfalls

1. **mDNS TXT keys.** Home Assistant's RAOP integration is picky. The
   `_raop._tcp` instance name must be `<MAC-clean>@<Name>._raop._tcp.local.`
   and the TXT record should include (per shairport-sync conventions):
   `ch=2 cn=0,1 da=true et=0,3 md=0,1 pw=false sv=false tp=UDP vs=103.2210.1
   am=ESP32 sf=0x4` (values **[VERIFY]** against what squeezelite-esp32
   advertises — that set is proven with HA). Missing `et` (encryption type)
   or wrong `tp` (transport) is the classic "device shows but fails to play".
2. **Timing/NTP channel.** AirPlay 1 uses the timing channel (UDP 319/320-style
   NTP timestamps on the agreed port) for clock sync. Many naive receivers
   skip it and get silent dropouts after ~30 s when the sender re-syncs. Do
   not stub the timing responder; answer timing requests with `esp_timer`
   -derived NTP-format timestamps.
3. **ALAC 44.1 kHz only.** AirPlay 1 senders always send 44.1 kHz/16-bit/stereo
   ALAC (or PCM with `md=1`). The I2S output is configured to exactly this —
   there is no resampler in the plan. If a sender ever negotiates otherwise,
   reject at RTSP rather than playing garbage.
4. **Audio sink backpressure.** `audio_write()` must be blocking (queue with
   timeout) or the RTP loop drops frames. Report drops upward; do not let the
   decode task busy-spin on a full I2S DMA queue.
5. **Session teardown.** Senders disconnect abruptly (phone locked, AirPlay
   switched). Must reap RTSP sessions on socket EOF and a session watchdog;
   leaked sockets/tasks on an ESP32 are fatal within a few sessions.
6. **`SO_REUSEADDR` + port reuse** across reconnects — lwIP honors it, but
   ensure the shim sets it on all three RTP sockets or the second session
   fails to bind.
7. **AES decrypt alignment.** The ALAC frames arrive AES-CBC encrypted per
   packet; decode in place if possible to avoid doubling the frame buffer.

## Integration checklist (ordered)

1. **[VERIFY]** Clone/inspect upstream:
   - `git clone https://github.com/philippe44/RAOP-Player`
   - `git clone https://github.com/sle118/squeezelite-esp32`
   Confirm actual file lists, licenses, and mbedTLS support in both.
2. Decide source of truth: **RAOP-Player `raop/` + `alac/`** (LGPL path,
   preferred) with squeezelite-esp32 as the porting reference. Record the
   upstream commit hash in `firmware/components/raop/VENDORING.md`.
3. Copy upstream files into `firmware/components/raop/upstream/` **unmodified**
   (keep every license/copyright header; add nothing to those files).
4. Write the shim: `esp_raop_platform.c/.h` (sockets, threads, mutex, clock,
   heap) implementing whatever small platform header upstream expects.
5. Wire the audio sink: upstream's audio-output callback → `audio_write()`
   (`firmware/main/audio.h`). Blocking semantics, int16 interleaved stereo.
6. mbedTLS glue for the RAOP RSA/AES steps (replacing OpenSSL).
7. mDNS advertisement (`esp_mdns`) with the TXT keys above; verify Home
   Assistant sees the device.
8. RTSP/RTP bring-up with a real sender (iPhone/Mac `AirPlay` menu; debug with
   `raop_send`/`sendfile`-style tools from RAOP-Player if still present).
9. Tune jitter buffer size and task priorities; measure free heap and stack
   high-water marks; record numbers in this doc.
10. License hygiene before first public commit: add LICENSE/THIRD-PARTY
    notices covering LGPL RAOP code + ALAC copyright.

## Steps already done in-repo

- `firmware/components/raop/` skeleton created: `CMakeLists.txt`,
  `VENDORING.md` (upstream copy instructions), `include/raop_sink.h` and
  `src/` with an ESP-IDF-native skeleton (RTP channel structs, NTP timestamp
  helpers, session lifecycle, stub protocol handlers) marked
  `// TODO(vendor):` wherever real upstream code must replace it. No crypto
  is implemented and **AirPlay does not work yet**.
