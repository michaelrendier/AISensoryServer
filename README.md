# AISensoryServer

An external sensory bank built on a **Samsung Galaxy A51 5G**, using [Phyphox](https://phyphox.org/) to handle all low-level sensor integration. The result is a single `live_state.json` file — one entry per sensor, always overwritten with the current value — that any AI or local service can poll without managing a stream.

Cameras are explicitly excluded. This is a *physical world state* feed, not a vision feed.

---

## Hardware: Samsung Galaxy A51 5G Sensor Inventory

| Sensor | Axes / Output | Phyphox Experiment |
|---|---|---|
| Accelerometer | x, y, z (m/s²) | Acceleration (without g) |
| Gyroscope | x, y, z (rad/s) | Gyroscope |
| Magnetometer | x, y, z (µT) | Magnetometer |
| Barometer | pressure (hPa) | Pressure |
| Ambient Light | lux (lx) | Light |
| Proximity | distance / near boolean (cm) | Proximity |
| GPS / GNSS | lat, lon, alt, speed, bearing, accuracy | GPS |
| Microphone — Primary (bottom) | amplitude (Pa), peak frequency (Hz) | Audio Amplitude / Audio Spectrum |
| Microphone — Secondary (top) | amplitude (Pa), peak frequency (Hz) | Audio Amplitude / Audio Spectrum |
| **Audio TDOA** | delay (ms), azimuth (°), foreground mic | derived — see Dual Mic section |

> **Step counter, linear acceleration, rotation vector, and gravity** are available as derived sensors in Phyphox and can be added to `live_state.json` with no hardware changes.

---

## Dual Microphone: Speaker Identification

The A51 5G places its two microphones approximately **140 mm apart** — primary at the bottom, secondary at the top. At the speed of sound (343 m/s), that separation yields a maximum **Time Difference of Arrival (TDOA) of ~0.41 ms**.

### How it works

1. **Two simultaneous Phyphox audio experiments** are configured — one for each Android audio source:
   - `VOICE_RECOGNITION` → bottom mic (primary, speech-optimised)
   - `CAMCORDER` → top mic (secondary, used during video recording, physically isolated)
2. The poller reads the most recent audio buffer from both at each tick.
3. **Cross-correlation** of the two amplitude envelopes gives the inter-mic delay `Δt`.
4. Azimuth is estimated as `θ = arcsin(Δt · c / d)` where `c = 343 m/s` and `d = 0.140 m`.
5. A foreground speaker held at <1 m will produce a consistent `Δt` while background conversation will fluctuate — the **variance of `Δt` over a rolling 500 ms window** is the primary discriminator.

### Practical limitations

- Phyphox exposes audio through its built-in experiment types; accessing both Android audio sources simultaneously requires **two concurrent Phyphox experiment sessions** running in separate browser tabs via the remote API.
- TDOA discrimination is angular only (a cone, not a point). Combine with amplitude ratio `A_primary / A_secondary` for coarse distance estimation.
- The system does **not** do voice identification — it identifies *which direction* sound is arriving from and whether that direction is stable (foreground speaker) or scattered (background noise).

---

## `live_state.json` Schema

```json
{
  "timestamp": "2026-05-26T10:30:00.123Z",
  "device": "Samsung Galaxy A51 5G",
  "poll_interval_ms": 250,
  "sensors": {
    "accelerometer":        { "x": 0.12,  "y": -9.78, "z": 0.34,  "unit": "m/s²",  "ts": 1748254200.123 },
    "gyroscope":            { "x": 0.001, "y": -0.002,"z": 0.000, "unit": "rad/s", "ts": 1748254200.123 },
    "magnetometer":         { "x": 24.5,  "y": -18.2, "z": 42.1,  "unit": "µT",    "ts": 1748254200.123 },
    "barometer":            { "pressure": 1013.25,                 "unit": "hPa",   "ts": 1748254200.123 },
    "light":                { "lux": 350.0,                        "unit": "lx",    "ts": 1748254200.123 },
    "proximity":            { "distance": 5.0, "near": false,      "unit": "cm",    "ts": 1748254200.123 },
    "gps": {
      "lat": 0.0, "lon": 0.0, "altitude": 0.0,
      "speed": 0.0, "bearing": 0.0, "accuracy": 0.0,
      "unit": "deg/deg/m/m·s⁻¹/deg/m",              "ts": 1748254200.123
    },
    "microphone_primary":   { "amplitude": 0.0023, "peak_hz": 320.0, "unit": "Pa/Hz", "ts": 1748254200.123 },
    "microphone_secondary": { "amplitude": 0.0018, "peak_hz": 318.0, "unit": "Pa/Hz", "ts": 1748254200.123 },
    "audio_tdoa": {
      "delay_ms": 0.18,
      "azimuth_deg": 22.5,
      "foreground_mic": "primary",
      "foreground_confidence": 0.87,
      "ts": 1748254200.123
    }
  }
}
```

The file is **always overwritten in place** — no history, no stream. Consumers treat it as a memory-mapped register bank.

---

## Architecture

```
┌─────────────────────────────┐
│   Samsung Galaxy A51 5G     │
│                             │
│  Phyphox (HTTP server :8080)│
│  ├── Accelerometer exp      │
│  ├── Gyroscope exp          │
│  ├── Magnetometer exp       │
│  ├── Pressure exp           │
│  ├── Light exp              │
│  ├── Proximity exp          │
│  ├── GPS exp                │
│  ├── Audio (bottom mic)     │
│  └── Audio (top mic)        │
└────────────┬────────────────┘
             │  WiFi / USB tether
             │  HTTP GET /get?<buffer>
┌────────────▼────────────────┐
│   poller.py  (host machine) │
│                             │
│  polls each experiment      │
│  cross-correlates audio     │
│  writes live_state.json     │
└────────────┬────────────────┘
             │
┌────────────▼────────────────┐
│   AI / consumer service     │
│   reads live_state.json     │
│   (file poll or inotify)    │
└─────────────────────────────┘
```

---

## Path to Execution

### Step 1 — Phone

1. Install **Phyphox** from the Google Play Store.
2. Create (or load) experiments for each sensor listed in the inventory table above.  
   Phyphox ships with built-in experiments for all of them except the secondary mic, which requires a custom experiment XML (see `experiments/` directory — to be added).
3. Start all experiments.
4. In Phyphox: tap the three-dot menu → **Allow Remote Access** → note the URL shown (e.g. `http://192.168.1.42:8080`).
5. Keep the screen on (`Settings → Developer options → Stay awake`).

### Step 2 — Host machine

```bash
git clone https://github.com/michaelrendier/AISensoryServer.git
cd AISensoryServer
pip install -r requirements.txt   # requests, numpy (for cross-correlation)
cp config.example.json config.json
```

Edit `config.json`:

```json
{
  "phyphox_host": "192.168.1.42",
  "phyphox_port": 8080,
  "poll_interval_ms": 250,
  "output_path": "live_state.json",
  "tdoa_window_ms": 500
}
```

### Step 3 — Run

```bash
python poller.py
```

`live_state.json` will appear in the repo root and update at the configured interval. Point any consumer at that file.

### Step 4 — Consume

```python
import json, time, pathlib

STATE = pathlib.Path("live_state.json")

while True:
    state = json.loads(STATE.read_text())
    az = state["sensors"]["audio_tdoa"]["azimuth_deg"]
    print(f"Foreground speaker at {az:.1f}°")
    time.sleep(0.25)
```

Or use `inotify` / `watchdog` to trigger on file change rather than polling.

---

## Roadmap

- [ ] `poller.py` — core polling loop
- [ ] `config.example.json`
- [ ] `experiments/dual_mic_secondary.phyphox` — custom XML for top mic via `CAMCORDER` source
- [ ] `experiments/` — exportable Phyphox experiment files for every sensor
- [ ] TDOA cross-correlation module
- [ ] `requirements.txt`
- [ ] Optional: WebSocket broadcast of `live_state.json` for remote consumers
- [ ] Optional: iOS port (Phyphox is available on App Store)

---

## Notes

- Phyphox's HTTP API returns JSON. Each buffer endpoint is `GET http://<host>:<port>/get?<buffer_name>` where `buffer_name` matches the internal experiment buffer (visible in the experiment's XML).
- The `updateMode: "single"` buffer type in Phyphox always holds the most recent sample — exactly the semantics this project needs.
- GPS will be `null` indoors or when the phone has no fix; consumers must handle absent fields gracefully.
- Phyphox runs experiments one at a time per session by default. To run multiple simultaneously, open each experiment in a **separate browser tab** pointed at the Phyphox remote server — each tab drives its own experiment independently.
