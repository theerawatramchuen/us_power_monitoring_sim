# Ultra Sonic Power Monitoring Simulator — UTAC AUTOMATION

A single-file browser simulator (`ultrasonic_power_simulator.html`) for an
ultrasonic-bonder power monitoring instrument. It simulates live voltage and
current waveforms, derives **power = V × I (positive only)**, extracts peak
envelopes, checks per-time-range **power** control limits, raises alarms, and
persists the control panel. This document is for developers extending it
toward a real DSO / DAQ-connected instrument.
<img width="890" height="2011" alt="image" src="https://github.com/user-attachments/assets/993fa0b1-4562-4599-93e8-81c06a0a3fc0" />

## 1. What this is today

Everything runs client-side in one HTML file — no backend, no build step.
"Live" data is faked: a base capture (voltage + current, 1000 points) is
re-randomized on a timer. The goal is to validate the **operator workflow and
alarm logic** before wiring up hardware, so the control-panel semantics should
carry over unchanged to the real instrument.

Display hierarchy:

- **Main chart:** Ultrasonic Power (W) — raw power (optional), positive-peak
  envelope, limit lines/zones, violation markers.
- **Minor charts:** Voltage (V), Current (A), Impedance (Ω). All four share one
  time axis, one zoom/pan state.

## 2. Signal model

| Signal | Definition |
|---|---|
| Current `I` (A) | CSV column `CH1` × `iScale` (default 0.1, same as `plot_waveform*.py`) |
| Voltage `V` (V) | CSV column `CH2` (falls back to `CH1`) × `vScale` (default 1.0 — **set to your probe ratio**) |
| Power `P` (W) | `V × I` per sample, negatives removed (see below) |
| Positive peak | rolling max, centered window `N` samples, `min_periods=1` (port of `compute_peak_envelope()` in `plot_waveform_elvp.py`) |
| Impedance `Z` (Ω) | `posPeak(V) / posPeak(I)`; `NaN` where either peak is < 5 % of its maximum (ramps) |

"Only positive" power is handled by `powerOf()`: the default clamps negative
products to 0; the Base Capture tab also offers absolute value. Decide which
matches your instrument's definition (clamp discards reactive power returns;
abs counts them as dissipation) and delete the option you don't want.

## 3. Architecture

```
Data source (swap for real HW)                 → srcI / srcV  (raw columns + time)
  genDefault() / parseCSV() / resample()       → rebuildAll() → baseTime, baseI, baseV
        │
        ▼
tick()  — every "update interval" ms
  • per-range random factors for I and V (simRows) + point noise  → liveI, liveV
  • recomputeDerived(): liveP, posPeakP, V/I envelopes, zImp
  • evaluateAlarms(true): posPeakP vs limitRows                    → alarmLog
        │
        ▼
render() → charts[]  (cvP power, cvV, cvI, cvZ — shared view.xMin/xMax)
```

Key state (top-level IIFE in the `<script>`):

| Variable | Meaning |
|---|---|
| `baseTime[]`, `baseI[]`, `baseV[]` | Base capture on a common time base (A, V) |
| `liveI[]`, `liveV[]`, `liveP[]` | Current simulated/measured waveforms |
| `posPeakP[]` | Positive-peak envelope of power — **the only trace checked against limits** |
| `limitRows[]` | `{id,start,end,max,min}` in **watts** |
| `simRows[]` | `{id,start,end,iMin,iMax,vMin,vMax}` — current/voltage amplification ranges |
| `view` | shared zoom/pan viewport |
| `alarmLog[]` | newest-first violation history (capped at 100) |

### Alarm rule

For each `limitRows` row, per evaluation, find the **worst** `posPeakP` sample
inside `[start, end]`: `HIGH` if it exceeds `max`, `LOW` if it is below `min`.
One alarm per bound per range per evaluation — not per sample (that floods the
log). `evaluateAlarms(false)` re-checks without logging; it runs immediately
when limits, envelope window, or negative-power mode change while running.
Voltage, current, and impedance have **no** limits by design.

## 4. Path to a real instrument

The swap is localized: **replace the data source, keep everything downstream.**

### 4.1 Replace the data source

`genDefault()` and the two CSV inputs are the only places data is
manufactured. A browser cannot speak SCPI/VISA/serial/Modbus directly, so a
small bridge process is required:

1. Bridge reads the instrument (SCPI via `pyvisa`/VXI-11 for a DSO; NI-DAQmx,
   Modbus, OPC-UA, etc. for a DAQ/PLC) — **both channels from the same
   acquisition** so V and I are sample-aligned.
2. Bridge pushes frames to the page over WebSocket, e.g.

   ```json
   { "time": [0.0000, 0.0001, ...], "ch1_raw": [...], "ch2_raw": [...] }
   ```
3. In the page, replace `setInterval(tick, ms)` with a `WebSocket.onmessage`
   handler that sets `baseTime`, `rawI`, `rawV`, calls `rebuildScaled()`-style
   scaling, then runs `recomputeDerived(); evaluateAlarms(true); render();`.
   Keep `iScale` / `vScale` client-side so calibration stays in the control
   panel.

**Alignment matters.** Power is a sample-by-sample product. If V and I come
from separate captures, the CSV path resamples V onto the current time base
(`resample()`), but any real timing skew between channels (probe/cable delay,
different trigger) directly corrupts P and Z. Calibrate deskew in the DSO or
bridge.

**Resolution matters.** The synthetic default is a 100 µs/sample envelope-like
signal. A real ultrasonic carrier (tens of kHz) needs the capture/sample rate
to resolve it; if your instrument outputs already-processed values (RMS/peak
envelopes), decide whether `V × I` on those is the quantity you actually want
(true instantaneous power vs. apparent power).

### 4.2 Remove or repurpose the "Waveform Simulation" tab

`simRows[]`, the per-tick random factors, and point noise only fabricate
motion. On hardware, delete that code path (or keep it behind a
"simulate / replay" mode toggle).

### 4.3 Alarm output beyond the browser tab

Alarms currently change on-screen state only. A real instrument also needs:

- a physical annunciator (tower light, buzzer, relay) driven by the **bridge**,
  which should evaluate the same `limitRows` rule independently;
- durable logging (file/DB/MES) for traceability and SPC;
- any process action (reject part, stop next cycle) routed through the
  bonder's existing safety/interlock system — a browser tab must not be the
  sole safety-relevant component.

### 4.4 Settings persistence

`saveConfig` / `loadConfigRaw` / `clearConfig` use Anthropic's in-artifact
`window.storage` when present, else IndexedDB (`IDB_NAME`). Outside Claude.ai
drop the `window.storage` branch, or point these three functions at a backend
(`/api/panel-settings`) so limits are shared across stations. Namespace
`SETTINGS_KEY` per station/line if needed. Saved config (`version: 2`)
contains limit rows, simulation rows, interval, noise, envelope window, raw
toggle, `iScale`, `vScale`, and negative-power mode. The key differs from the
earlier current-monitor build, so old saved settings are not loaded.

## 5. Extension points

| Want to change... | Look at... |
|---|---|
| How power is formed / negative handling | `powerOf()` |
| Impedance definition or masking threshold | `recomputeDerived()` |
| Envelope calculation | `computePeakEnvelope()` |
| Alarm rule | `evaluateAlarms()` |
| Add / remove a minor chart | `charts[]` array (+ a canvas in the HTML) |
| Axis scaling | `niceAxis()`, `powerAxis()`, `envAxis()`, `impedanceAxis()` |
| CSV quirks (metadata rows, alt column names) | `parseCSV(text, candidates)` — mirrors `load_waveform()` in the Python scripts |
| Row numbering | `renderLimitsTable()` / `renderSimTable()` — `#` is `idx+1`, independent of internal `row.id` |

## 6. Known simplifications

- Alarm evaluation is O(rows × samples) per tick and the envelope is
  O(samples × window) per channel — fine at 1000 points; switch to a
  monotonic-deque rolling max/min if capture sizes grow by orders of magnitude.
- Impedance is a peak-envelope ratio, not a phase-aware complex impedance.
- No auth, multi-user, or multi-instrument concept; one tab, one instrument.
- No unit tests ship with the page. At minimum add tests for
  `computePeakEnvelope` (against pandas output) and `evaluateAlarms`
  boundary conditions at range edges.
- The synthetic default waveform is illustrative only; don't tune real limits
  against it.

## 7. File map

```
ultrasonic_power_simulator.html   — the entire app (HTML + CSS + JS, no deps)
README.md                         — this file
```
