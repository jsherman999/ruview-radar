# ruview-radar — Plan for a simple implementation of RuView

Status: **plan only**, nothing implemented yet.
Upstream reviewed: [`ruvnet/RuView`](https://github.com/ruvnet/RuView) (MIT), commit `48f8091` (2026-11-11).
Unless stated otherwise, every `path:line` below is relative to that upstream commit.

---

## 1. Goal

Build a small, readable, single-process app. It takes the WiFi Channel State
Information (CSI) that an ESP32-S3 running **RuView's existing firmware** already
streams. From it the app works out four things for each sensor node:

- presence;
- motion level;
- breathing rate;
- heart rate, as an experimental, clearly-labelled extra.

It shows the results live on a radar-style web page.

The app should run on a laptop or Raspberry Pi with `pip install` and one command. It should
be testable end to end with **no hardware**, using a correct synthetic CSI source and
record/replay.

### Non-goals (v1)

- Pose estimation, pretrained models, embeddings and LoRA.
- Multistatic mesh, TDM or channel hopping.
- Passive-radar range-Doppler.
- Fall detection and person counting.
- Home Assistant, Matter or HomeKit.
- MCP or agent tooling, witness chains, the cog/plugin catalog.
- Writing or modifying firmware.

§3 says why each one is out.

---

## 2. What RuView is: review summary

RuView is a very large platform. Its main parts:

- a Rust workspace with 73 crates in `v2/crates/`;
- ESP-IDF firmware for ESP32-S3/C6 in `firmware/esp32-csi-node/`;
- a deprecated Python reference pipeline in `archive/v1/`;
- npm harnesses, a Claude Code plugin, smart-home bridges, 324 ADR files, and a 105-module edge "cog" catalog.

The useful core is much smaller than the surface area.

```
ESP32-S3 (RuView firmware)
  ├─ raw CSI frames   UDP :5005  magic 0xC5110001  (20-byte header + int8 I/Q)
  └─ edge vitals      UDP :5005  magic 0xC5110002  (32 bytes, ~1 Hz, computed on-device)
        │
        ▼
Rust sensing-server (v2/crates/wifi-densepose-sensing-server)
  parse → amplitude/phase → feature extraction → motion/presence classification
        → FFT vitals (breathing / heart rate) → JSON over WebSocket /ws/sensing
        │
        ▼
Browser UI (ui/, vanilla JS + three.js)
```

### 2.1 What is real vs. what is claimed

The upstream README is candid about maturity. A simple version should only aim at the
parts that actually work.

| Capability | Upstream status | Source |
|---|---|---|
| CSI capture + UDP streaming from ESP32-S3 | Real, hardware-validated | `firmware/esp32-csi-node/main/csi_collector.c:170-293` |
| Presence / motion from CSI variance | Real, heuristic DSP | `edge_processing.c:1190-1220`, `sensing-server/src/csi.rs:536-689` |
| Breathing / heart rate | Heuristic DSP; HR especially unreliable on a single ESP32 | `docs/user-guide.md:1466` |
| 17-keypoint pose on a single ESP32 | First-cut: PCK@20 = 3.0%; runtime is a stub returning `confidence=0` | `README.md:161`, `README.md:339` |
| "82.3% presence" | Temporal-triplet accuracy of an embedding model, **not** occupancy accuracy | `README.md:139` |
| Through-wall range ~5 m | Stated, no measurement cited | `docs/user-guide.md:2961` |
| mmWave fusion | ADR **Proposed**; firmware UART parsers exist | `docs/adr/ADR-063-mmwave-sensor-fusion.md:3`, `firmware/.../mmwave_sensor.c` |

**Implication:** a simple implementation can credibly deliver presence, motion and
breathing-while-still. Heart rate is an experimental extra. Everything else is research.

---

## 3. Keep / drop

| RuView feature | v1 decision | Why |
|---|---|---|
| ESP32 firmware | **Reuse unchanged** (flash an upstream release) | Hardest part, already works; zero firmware code for us |
| ADR-018 raw CSI frame decode | **Keep** | The core input |
| `0xC5110002` edge-vitals packet decode | **Keep** (display as "device-reported, unvalidated") | Free cross-check, 30 lines |
| Host presence/motion | **Keep**, simplified | Core value |
| Breathing (FFT peak, 0.1–0.5 Hz) | **Keep** | Works when subject is still |
| Heart rate (0.8–2.0 Hz) | **Keep, experimental, off by default in UI** | Weak on single-antenna CSI |
| Fall detection | Drop | Default threshold appears unreachable (§8.2); no validated algorithm |
| Person counting | Drop | Heuristic subcarrier-group clustering, unvalidated |
| Pose / DensePose / models | Drop | First-cut, stub runtime (§2.1) |
| Passive radar range-Doppler (`scripts/passive-radar.js`) | Drop | Uncalibrated phase slopes, APs matched by channel only (`passive-radar.js:288-296`) |
| Multi-node mesh, TDM, channel hopping | Drop (multi-node *display* is a stretch goal) | Complexity |
| RSSI-only laptop mode | Stretch | Coarse; upstream server has no Linux RSSI source (`docs/user-guide.md:357-361`) |
| mmWave (LD2410 / MR60BHA2) | Stretch: gives a *real* range for the radar | Firmware already emits fused packet `0xC5110004` |
| HA / Matter / HomeKit / MCP / cogs / witness | Drop | Out of scope |

---

## 4. Architecture

**Stack:** Python ≥ 3.11 with two runtime dependencies, `numpy` and `aiohttp`. The
front end is one static page using vanilla JS and Canvas 2D: no build step, no CDN.
Config is TOML, read with the standard-library `tomllib`.

```
                ┌──────────────────── ruview-radar (one process, asyncio) ──────────────────┐
 ESP32 ──UDP──► │ sources.py  ──►  protocol.py  ──►  detector.py (per node)  ──►  server.py │ ──WS──► browser
 synth  ──────► │  udp | synth |     decode/          ring buffer, rate est.,      aiohttp: │        radar.js
 replay ──────► │  replay          validate          calibration, presence,      /ws, REST,│
                │                                    breathing, HR (exp.)         static    │
                └───────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Repository layout

```
ruview-radar/
  PLAN.md                    this file
  README.md
  pyproject.toml             numpy, aiohttp; dev: pytest, pytest-aiohttp, ruff
  ruview-radar.example.toml
  src/ruview_radar/
    protocol.py              ADR-018 + edge-vitals encode/decode, strict validation
    synth.py                 synthetic CSI scene model (empty / breathing / walking)
    sources.py               UdpSource, ReplaySource, SynthSource → async iterator of Frame
    dsp.py                   hampel, rate estimation, resampling, top-K, spectral peak
    detector.py              NodeDetector: calibration, presence hysteresis, vitals
    server.py                aiohttp app: GET /, /ws, /api/state, POST /api/nodes/{id}/calibrate
    cli.py                   `ruview-radar serve | synth | record | replay`
    web/index.html, radar.js, style.css
  tests/
    test_protocol.py  test_dsp.py  test_detector.py  test_server.py
    fixtures/                small SYNTHETIC recordings only (never real CSI/person data)
  tools/eval_recording.py    MEASURED evaluation against labelled recordings
  docs/hardware-log.md       boot logs + measured results from real silicon
  .github/workflows/ci.yml   ruff + pytest on 3.11/3.12
```

### 4.2 CLI

```bash
ruview-radar serve  --source udp    --udp-port 5005 --http 127.0.0.1:8080   # live ESP32
ruview-radar serve  --source synth  --scene breathing                        # no hardware
ruview-radar serve  --source replay --file rec.csi.jsonl --speed 1.0
ruview-radar synth  --host 127.0.0.1 --port 5005 --scene walking-after:20    # UDP emitter
ruview-radar record --udp-port 5005 --seconds 120 --out data/rec.csi.jsonl
```

---

## 5. Data contracts

### 5.1 Inputs (UDP, little-endian)

**Raw CSI, `0xC5110001`.** Layout taken from `csi_collector.c:197-280` and confirmed
against `v2/crates/wifi-densepose-hardware/src/esp32_parser.rs:168-245`. Python struct
format: `"<IBBHIIbbBB"`, 20 bytes.

| Off | Type | Field | Notes |
|---|---|---|---|
| 0 | u32 | magic | `0xC5110001` |
| 4 | u8 | node_id | |
| 5 | u8 | n_antennas | accept 1–4 |
| 6 | u16 | n_subcarriers | per antenna |
| 8 | u32 | freq_mhz | e.g. 2412 |
| 12 | u32 | sequence | detect drops/reorder |
| 16 | i8 | rssi dBm | |
| 17 | i8 | noise_floor dBm | |
| 18 | u8 | ppdu_type | 0 HT/legacy, 1 HE-SU, 2 HE-MU, 3 HE-TB, 0xFF unknown |
| 19 | u8 | flags | bit0 40 MHz, bit2 STBC, bit4 sync valid, **bit5 first word zeroed** |
| 20 | i8×2×N | I/Q pairs | N = n_antennas × n_subcarriers; ≤ 1024 pairs |

**Edge vitals, `0xC5110002`.** 32 bytes, layout from `edge_processing.h:212-227`.
Host decode follows `sensing-server/src/csi.rs:161-195`. Python struct format:
`"<IBBHIbB2xffII"`.

| Off | Type | Field | Decode |
|---|---|---|---|
| 4 | u8 | node_id | |
| 5 | u8 | flags | bit0 presence, bit1 fall, bit2 motion |
| 6 | u16 | breathing | ÷ 100 → BPM |
| 8 | u32 | heart rate | **÷ 10000** → BPM |
| 12 | i8 | rssi | |
| 13 | u8 | n_persons | |
| 16 | f32 | motion_energy | |
| 20 | f32 | presence_score | |
| 24 | u32 | timestamp_ms since boot | |

The firmware sends other magics: `0xC5110003` features, `0xC5110004` fused mmWave
vitals (48 bytes), `0xC5110005`–`0xC5110007`, and sync `0xC511A110`. These are
**counted and ignored** in v1, except `0xC5110004`, which the mmWave stretch goal uses.

**Validation rules.** Any datagram that breaks one of these is dropped and counted;
the parser never raises.

- The datagram length is exactly `20 + 2N`. For vitals it is exactly 32.
- `1 ≤ n_antennas ≤ 4`.
- `N ≤ 1024`.
- Bound to the configured UDP port.
- Optional `--udp-allow` source-IP allowlist.

### 5.2 Output: WebSocket `/ws`

The server pushes JSON at 5 Hz.

```json
{
  "type": "state", "ts": 1763000000.12, "source": "udp",
  "nodes": [{
    "node_id": 1, "label": "living room", "bearing_deg": 45,
    "rssi_dbm": -52, "rate_hz": 19.7, "n_subcarriers": 64, "age_s": 0.05,
    "drops": 3, "calibration": {"state": "done", "threshold": 0.012, "progress": 1.0},
    "motion": 0.031, "presence": true,
    "breathing": {"bpm": 14.2, "confidence": 0.71, "valid": true},
    "heart": {"bpm": null, "confidence": 0.0, "valid": false, "experimental": true},
    "edge": {"presence": true, "motion_energy": 0.04, "breathing_bpm": 14.0,
             "heart_bpm": 71.0, "age_s": 0.6, "label": "device-reported, unvalidated"},
    "history": {"motion": [0.01, 0.02, 0.03]}
  }],
  "stats": {"frames": 12034, "dropped_invalid": 2, "ignored_magic": 120}
}
```

**REST endpoints:**

- `GET /api/state` returns the same JSON as the WebSocket.
- `POST /api/nodes/{id}/calibrate {"seconds": 60}` starts an empty-room calibration.
- `GET /healthz`.

**Security defaults:**

- HTTP binds `127.0.0.1`.
- Binding to any other address requires `--token`, which the server checks as an
  `Authorization: Bearer` header or `?token=` on `/ws`.
- UDP binds `0.0.0.0:5005`, because the ESP32 must reach it. An allowlist is recommended.

---

## 6. Algorithms

Parameters come from upstream, and every frame keeps its **host arrival time**.

### 6.1 Per-frame preprocessing (`dsp.py`)

1. **Complex CSI.** `H = I + jQ` and amplitude `|H|`. Firmware and server compute it
   the same way (`csi.rs:316-321`).
2. **Antennas.** If `n_antennas > 1`, average amplitude across antennas. ESP32-S3
   reports 1.
3. **Subcarrier mask.**
   - If flag bit 5 is set, the firmware zeroed the first 4 bytes, so drop the first 2
     bins (`csi_collector.c:283-292`).
   - After 2 s, also drop null/guard bins with long-run mean amplitude < 2
     (`scripts/rf-scan.js:61-64`).
4. **Hampel filter.** Applied per subcarrier, over time, on amplitude: half-window 3,
   threshold 3σ with σ = 1.4826·MAD (`wifi-densepose-signal/src/hampel.rs:21-44`).
5. **Sample-rate estimate.** EMA of inter-arrival time, clamped to 8–60 Hz.
   - Nominal rates are 20 Hz on S3 and 8 Hz on C6 (`edge_processing.c:41-47`,
     `edge_processing.h:72-94`).
   - Do **not** assume 100 Hz. The `wifi-densepose` Python defaults do, which does not
     match the firmware.
6. **Ring buffer.** 30 s per node.
7. **Resampling.** Before any spectral step, resample linearly onto a uniform 20 Hz grid.

### 6.2 Motion and presence (`detector.py`)

- **Top-K.** Every 5 s, pick the K = 8 subcarriers with the highest amplitude variance
  over the last 10 s (`nvs_config.c:84`, `edge_processing.c:1129-1146`).
- **Motion.** `motion = median over top-K of std(amp[last 1 s]) / mean(amp[last 1 s])`.
  - This is the coefficient of variation, so it does not depend on scale.
  - It is amplitude-based on purpose: per-packet CFO/STO makes raw ESP32 phase noisy.
    Sanitised phase is a post-v1 option.
- **Calibration.** Run for a **wall-clock** period, default 60 s, minimum 30 s.
  - Done on demand, or automatically on first start with a banner saying "room must be
    empty".
  - Threshold `T = max(mean + 3σ, floor)` (`edge_processing.h:138-139`,
    `edge_processing.c:540-564`).
  - Persisted per node in `state/calibration.json`.
  - Upstream counts 1200 *frames*, which is 60 s at 20 Hz but 150 s at 8 Hz. We use seconds.
- **Hysteresis**, copied from firmware (`edge_processing.h:163-164`, `edge_processing.c:505-534`):
  - turn **on** when motion > T;
  - **hold** while motion ≥ 0.5·T;
  - turn **off** after 5 consecutive updates below 0.5·T, plus a minimum hold of 2 s.
- **Motion level** for the UI: `motion / T`, banded as absent < 1, still 1–2,
  moving 2–5, active > 5. These are display bands only, not claims.

### 6.3 Breathing (`dsp.py` + `detector.py`)

- **Input signal.** Mean amplitude over the top-K subcarriers, over a 30 s window.
  Detrend it, which removes the mean and a linear trend.
  - Upstream's server uses mean amplitude over a 30 s buffer (`vital_signs.rs:95-96`).
  - Post-v1 option: use the first principal component instead of the mean.
- **Spectrum.** Hann window, zero-pad to 4096, peak search in **0.1–0.5 Hz (6–30 BPM)**,
  parabolic interpolation (`vital_signs.rs:214-305`).
- **Confidence.** Peak-bin power ÷ total in-band power.
- **Publish only if all of these hold:**
  - presence is on;
  - motion is in the "still" band, because breathing is buried under gross motion;
  - the window is ≥ 80% filled;
  - confidence ≥ 0.55, the upstream threshold (`sensing-server/src/main.rs:8777`).
- **Update rate:** 1 Hz.

### 6.4 Heart rate (experimental)

- 15 s window, band 0.8–2.0 Hz (48–120 BPM), same FFT peak method.
- Reject candidates within 8% of k·f_breathing for k = 1..6 (`edge_processing.c:249-305`).
- Report the median of the last 13 valid estimates (`edge_processing.c:313-333`).
- Hidden behind a UI toggle and always labelled *experimental / unvalidated*.

### 6.5 Edge-vitals pass-through

- Decode `0xC5110002` and show it next to the host values, labelled
  "device-reported, unvalidated".
- No fusion in v1. Disagreement between the two is useful diagnostic information.

---

## 7. Radar UI (`web/`)

The display has to be honest about one physical fact: **a single TX→RX CSI link gives
no angle and no calibrated range**. So the design is:

- **Polar canvas** with a rotating sweep line, 4 s per revolution, and a phosphor-style
  decay trail. The sweep is cosmetic, and the legend says so.
- **Node wedges.**
  - Each node owns a wedge at a user-configured `bearing_deg` and width (TOML), for
    example "sensor by the sofa, looking NE".
  - Wedge glow intensity is `motion / T`. When presence is on, the wedge edge turns solid.
  - Rings are labelled **motion ×threshold (1×, 2×, 5×)**, not metres.
- **True-range blips (stretch, mmWave only).** When `0xC5110004` packets carry
  `mmwave_distance` (cm), draw a blip at that radius on the node's bearing, with rings
  relabelled in metres for that wedge.
- **Side panel, one card per node:**
  - presence badge;
  - motion sparkline (60 s);
  - breathing BPM with a confidence bar;
  - HR (toggle, "experimental");
  - RSSI, frame rate, drops, last-seen age;
  - calibration state and button;
  - edge-reported values in grey.
- **States:** "waiting for frames", "stale (> 3 s)", and "calibrating (n s left, keep
  room empty)".
- **Layout:** responsive down to phone width, light and dark themes, no external assets.

---

## 8. Upstream issues found during review (design around these)

1. **`scripts/synth-csi-udp.py` emits the wrong header.**
   - It packs `"<I B B B B H I b b I"` (`synth-csi-udp.py:24`): freq as **u16 at offset 8**
     and seq at **offset 10**.
   - Firmware and both Rust parsers use **u32 freq at 8** and **seq at 12**
     (`csi_collector.c:212-216`, `esp32_parser.rs:168-195`).
   - Result: parsed freq and seq are garbage and rssi/noise read 0. **Do not port it.**
     Our `synth.py` encodes through `protocol.py`, and a test pins the byte offsets.
2. **The default fall threshold appears unreachable.**
   - `unwrap_phase` bounds each per-frame step to ±π (`edge_processing.c:153-160`).
   - So `accel = |v − v_prev| ≤ 2π ≈ 6.28`, yet the default `fall_thresh` is 15.0
     (`nvs_config.c:73`, `edge_processing.c:1226-1254`).
   - This is inferred from code reading and not verified on hardware. It is one reason
     fall detection is out of v1.
3. **Ports don't agree.**
   - Firmware and server use UDP 5005 (`Kconfig.projbuild:41-49`).
   - The Node "radar" scripts default to 5006 (`scripts/passive-radar.js:38`).
   - Server HTTP/WS is 8080/8765 natively (`sensing-server/src/cli.rs:110-118`), but
     3000/3001 in Docker (`docker/docker-entrypoint.sh:157-158`).
   - We use one HTTP port that serves both the page and the WebSocket.
4. **Calibration length is counted in frames, not time.** 1200 frames is about 60 s on
   S3 but about 150 s on C6 (§6.2). We use seconds.
5. **Docs and code disagree about heart rate.**
   - The firmware header says heart rate is found by zero-crossing; the code uses
     autocorrelation (`edge_processing.c:13` vs `:249-305`).
   - HR bands differ between firmware (0.8–2.0 Hz, search 45–180 BPM) and server
     (0.667–2.0 Hz) (`vital_signs.rs:3` vs `:24`).
   - We choose 0.8–2.0 Hz and document it.
6. **Sample-rate assumptions conflict.** The firmware gives 20 Hz (S3) or 8 Hz (C6), but
   the Python wheel's `esp32_default()` assumes 100 Hz (`python/README.md`). We measure
   the rate from timestamps.
7. **I/Q byte order.**
   - RuView reads each pair as (I, Q) (`esp32_parser.rs:224-234`).
   - ESP-IDF documentation describes CSI entries as imaginary byte then real byte.
   - Amplitude is unaffected, and our features are amplitude-based, but phase-based
     extensions must check this.
8. **Upstream's simplest live viewer can't authenticate.**
   - `examples/through-wall/index.html` has no auth handling, so it fails against a
     token-gated server.
   - The README calls the original ESP32 "single-core" (`README.md:825`), but most
     original ESP32 parts are dual-core. Meanwhile the single-core C6 *is* supported.
   - Minor, but a reminder to treat upstream docs as claims and code as truth.

---

## 9. Milestones

Each milestone ends with green CI and a short demo note in the PR.

| # | Deliverable | Acceptance criteria | Size |
|---|---|---|---|
| **M0** | Skeleton: `pyproject.toml`, package, ruff, pytest, CI, `.gitignore` (`data/`, `state/`, `*.csi.jsonl`) | `pip install -e .[dev] && pytest` green in CI | 0.5 d |
| **M1** | `protocol.py`, `synth.py`, `sources.py`, `record` / `replay` / `synth` CLI | Golden-byte test pins every ADR-018 offset; vitals packet round-trips (BPM×100, ×10000); 10k random datagrams never raise; synth → UDP → record → replay is lossless | 1 d |
| **M2** | `dsp.py` + `detector.py`: rate estimation, Hampel, top-K, motion, calibration, hysteresis | **SYNTHETIC:** after a 60 s empty-room calibration, a 5 min empty scene gives 0 presence-on events (seeded); walking injected at t = 20 s turns presence on within ≤ 1.0 s; presence turns off within ≤ 5 s after motion stops; frame-rate jitter of 15–25 Hz and 5% drops don't change outcomes | 1.5 d |
| **M3** | Breathing + experimental HR | **SYNTHETIC:** a 0.25 Hz amplitude modulation at 2% depth plus noise reads **15 ± 1 BPM** within 30 s; sweep 8–25 BPM; HR 1.2 Hz → **72 ± 3 BPM** with a breathing harmonic present; no estimate is published while "moving" | 1.5 d |
| **M4** | `server.py` + radar UI | `ruview-radar serve --source synth` renders wedges, sweep, cards; WS schema test (aiohttp test client); 127.0.0.1 default; `--token` enforced off-localhost; screenshot at 390 px and 1280 px | 1.5 d |
| **M5** | Hardware bring-up and **MEASURED** baseline | Flash an upstream release to one ESP32-S3; provision `target_ip` / `target_port=5005` / `node_id` (`provision.py:509-573`); boot log committed to `docs/hardware-log.md`; three labelled 5 min recordings (empty / walking / seated-still); `tools/eval_recording.py` reports presence TPR/FPR and breathing error against a manual count, all tagged MEASURED with the exact command. Recordings themselves are **not** committed | 1–2 d |
| M6 (stretch) | Multi-node wedges · mmWave range blips via `0xC5110004` · Linux RSSI-only fallback (`iw`, ~2 Hz, coarse) · MQTT publish for Home Assistant | Each behind a flag with its own tests | — |

Rough total for M0–M5: **7–8 working days**. Hardware delivery is the critical path for M5.

---

## 10. Testing strategy

- **Unit tests.**
  - Protocol: golden bytes, bounds checks, truncation, wrong magic, oversize N.
  - DSP: Hampel against hand-computed outliers; spectral peak with parabolic
    interpolation on known sinusoids; resampler against an analytic signal.
- **Scenario tests.** `synth.py` builds scenes from a small physical-ish model, all
  seeded and deterministic:
  - a static multipath baseline per subcarrier;
  - breathing as a 1–3% amplitude modulation, a different phase per subcarrier;
  - walking as broadband random-walk amplitude perturbation;
  - Gaussian noise, rate jitter and packet drops.

  `detector.py` must meet the M2/M3 numbers on these scenes.
- **Reference fixture idea.** `archive/v1/data/proof/generate_reference_signal.py` builds
  3 antennas × 56 subcarriers at 100 Hz. It sums 5 multipath taps and adds a 0.3 Hz
  breathing and a 1.2 Hz motion modulation. Port that idea (not the code) as one more
  scene, and check that we recover 0.3 Hz.
- **Optional oracle (not a CI dependency).** Compare breathing estimates against the
  upstream PyPI wheel (`pip install wifi-densepose`, `BreathingExtractor`) on the same
  synthetic input, after setting its sample rate correctly.
- **Server tests.** `pytest-aiohttp`: `/api/state` schema, WS push cadence, token
  enforcement, `calibrate` endpoint.
- **Honesty labels.** Following upstream's own rule (upstream `CLAUDE.md`), any accuracy
  number in docs is tagged **MEASURED** (with a reproducer command), **CLAIMED** or
  **SYNTHETIC**. Synthetic test thresholds are never presented as real-world accuracy.

---

## 11. Privacy and safety

- This is not a medical device or a safety system. The README and UI footer say so.
- Never commit real CSI recordings, Wi-Fi credentials or provisioning output.
  `.gitignore` covers `data/`, `state/` and `*.csi.jsonl`. Only synthetic fixtures go
  in `tests/fixtures/`.
- The HTTP server is local-only by default. Exposing it requires a token.
- Every datagram is validated before use (§5.1).

---

## 12. Open questions / assumptions

These are assumed for now. Answers would change the plan as noted.

1. **Hardware.** Assumed: one ESP32-S3 dev board and a 2.4 GHz AP.
   - With 2–3 boards, promote multi-node wedges into M5.
   - With an LD2410 or MR60BHA2 mmWave module, promote range blips. That is the only
     path to a *true* radar display.
2. **Language.** Assumed: Python for readability and speed of iteration. A Rust port of
   `protocol.py`, `dsp.py` and `detector.py` is straightforward later if a Pi Zero-class
   target needs it.
3. **Host.** Assumed: a laptop or Raspberry Pi 4/5 on the same LAN as the ESP32.
4. **Firmware version.** M5 should pin a specific upstream firmware release tag and
   record it in `docs/hardware-log.md`. The release must stream raw `0xC5110001` frames
   to the host, not only edge packets. Check which `edge_tier` setting does that during
   bring-up.
