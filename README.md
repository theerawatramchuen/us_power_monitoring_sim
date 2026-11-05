# US Power Monitoring Controller — UTAC AUTOMATION

Application document · version 1.0.0

Software for the **US Power Monitoring Controller** (Windows PC). It measures ultrasonic **power** with a
**PicoScope 2204A**, shows it live in a web screen, checks it against operator-set limits and logs every failure.

| Scope input | Signal | Sensor |
|---|---|---|
| **CH1** | Ultrasonic **current** | Tektronix **CT1** AC current probe (into a 50 Ω terminator) |
| **CH2** | Ultrasonic **voltage** (AC) | Voltage probe (×10 assumed) |

> **Status.** Tested end to end with the built-in simulated scope (real Chromium browser: live frames, alarms, logging,
> settings, reconfiguration). The PicoScope driver code was tested against fake drivers and cross-checked against Pico's
> official Python wrapper. On a real 2204A it has been confirmed to **open the scope, configure it, capture (self-test) and
> start the generator**. It has **not been calibrated against a reference instrument**, and the trigger behaviour with a real
> bond signal has not been verified. Do the commissioning checklist (section 6) before trusting any reading.

---

## Contents

1. [What it does](#1-what-it-does)
2. [Architecture and files](#2-architecture-and-files)
3. [Hardware connection](#3-hardware-connection)
4. [Installation](#4-installation)
5. [Starting and stopping](#5-starting-and-stopping)
6. [Commissioning checklist](#6-commissioning-checklist)
7. [Using the screen](#7-using-the-screen)
8. [Signal processing and the alarm rule](#8-signal-processing-and-the-alarm-rule)
9. [Test signal from the scope generator (AWG)](#9-test-signal-from-the-scope-generator-awg)
10. [Files written](#10-files-written)
11. [Settings reference](#11-settings-reference)
12. [Command-line options and HTTP API](#12-command-line-options-and-http-api)
13. [Limits of this design](#13-limits-of-this-design)
14. [Extending the application](#14-extending-the-application)
15. [Tests](#15-tests)
16. [Troubleshooting](#16-troubleshooting)
17. [Sources](#17-sources)

---

## 1. What it does

For every bond cycle the scope captures both channels at the same instants. The controller:

1. converts them to amperes and volts (CT1 sensitivity, probe ratio, optional polarity inversion);
2. forms **power = V × I**, kept positive;
3. takes the **rolling positive-peak envelope** of power, voltage and current (same method as the original `plot_waveform_elvp.py`);
4. checks the power envelope against the **power limits per time range** (relative to the trigger) and raises an alarm with the failure details;
5. logs each alarm to a CSV file and keeps the capture that caused it;
6. shows everything live in a browser: **Power (W)** is the main chart, **Voltage, Current and Impedance** are the minor charts.

The same screen also runs **stand-alone as a browser simulator** (open `web/index.html` as a file) with synthetic or CSV-based waveforms, so limits can be prepared without hardware.

## 2. Architecture and files

```
 CT1 ─► 50 Ω ─► CH1 ┐                                          ┌─ browser (this PC, or another PC on the LAN)
                    ├─ PicoScope 2204A ─ USB ─► app.py ─ HTTP/WebSocket ─┤   power + V / I / Z charts, limits, alarms
 voltage probe ─► CH2 ┘   (block capture,        │                       └─ operator settings stored on the controller
                           hardware trigger)     ├─ logs/alarms_YYYY-MM-DD.csv
                                                 ├─ captures/capture_*.csv   (TIME,CH1,CH2)
                                                 └─ data/panel_settings.json
```

| File | Role |
|---|---|
| `app.py` | Entry point. Starts the acquisition thread and the web server (aiohttp HTTP + WebSocket), alarm logging, capture saving, self-test, command-line options |
| `acquisition.py` | Thread that owns the scope: open → configure → arm → wait for trigger → read → scale → publish. Reconnects automatically after a fault |
| `common.py` | Data structures and `scale_capture` (ADC volts → amperes/volts) |
| `config.py` | Default settings, validation, safe atomic saving of `config.json` |
| `scope_pico2000.py` | Driver for the **PicoScope 2204A** (original `ps2000` API via `picosdk`) and the optional test-signal generator |
| `scope_pico.py` | Driver for newer 2000-series scopes (2205 MSO, 2206/2207/2208; `ps2000a` API) |
| `scope_auto.py` | `driver: "auto"` — tries `ps2000` first, then `ps2000a` |
| `scope_sim.py` | Simulated scope for demos and testing (`--simulate`) |
| `diagnose.py` | `--diagnose`: finds out why the scope cannot be opened |
| `awg_play.py` | Stand-alone player of a DSO CSV on the scope's generator output |
| `web/index.html` | The monitor screen (single file; also works stand-alone as the simulator) |
| `config.json` | Instrument, server and logging settings |
| `start_app.bat` | Starts the controller from an existing `env` folder |
| `start_monitor.bat` | First-run installer + launcher (creates `.venv`) |
| `requirements.txt` | `aiohttp`, `numpy`, `picosdk` |
| `tek0001.csv` | Example DSO capture used by the test generator |
| `tests/` | Fake-driver tests |

> **Driver families.** The 2204A and 2205A use the **original `ps2000`** driver; the 2205 MSO and 2206/2207/2208 use **`ps2000a`**.
> A 2204A is invisible to `ps2000a` (`PICO_NOT_FOUND`). The default `driver: "auto"` handles this.

## 3. Hardware connection

1. **CH1 — CT1.** The CT1 is specified at **5 mV/mA (= 5 V/A) only into a 50 Ω load**. The 2204A input is 1 MΩ, so fit a
   **50 Ω feed-through terminator** at the CH1 BNC. Clamp the CT1 around the single conductor carrying the transducer current
   (arrow in the direction of conventional current; if power reads negative or zero, tick *Invert CH1*).
   CT1 limits (Tektronix datasheet): 12 A peak, **0.5 A RMS max**, 1 A·s, usable 25 kHz – 1 GHz (no DC response).
2. **CH2 — voltage.** The 2204A input range tops out at ±20 V, so use a **×10 probe** for transducer drive voltages and
   keep *Voltage probe ratio* = 10 (use 1 for a ×1 probe).
3. **Grounds.** The scope ground is tied to the PC/USB ground and shared by both channels. Connect the probe ground clip only to a
   ground-referenced point; if the generator output floats, use an isolated/differential probe instead.
4. **USB** cable to the controller PC. **Only one program can use the scope at a time** — close PicoScope 6/7 completely
   (it starts by itself when the scope is plugged in; check the system tray) and do not run `awg_play.py` together with `app.py`.

## 4. Installation

1. Install **Python 3.10+ (64-bit)** from python.org (tick *Add python.exe to PATH*; keep the `py` launcher). Development used Python 3.12.
2. Install the **64-bit PicoSDK** (or *PicoScope 7*) from picotech.com. It provides the drivers; the 2204A uses `ps2000.dll`.
3. Copy this folder to the PC (e.g. `D:\us_power_controller`).
4. Create the Python environment once and install the packages (internet needed once):
   ```
   cd D:\us_power_controller
   python -m venv env
   env\Scripts\activate
   pip install -r requirements.txt
   ```
   (`start_monitor.bat` does the same into a `.venv` folder automatically on its first run.)
5. Check the scope: `python app.py --diagnose` (section 16). Look for `OPENED OK` under the `ps2000` section.

## 5. Starting and stopping

* **Double-click `start_app.bat`** — goes to its own folder, activates `env`, runs `python app.py`. Extra options are passed on
  (e.g. `start_app.bat --diagnose`). The window stays open after the program stops so any error can be read.
  If `env` does not exist it says how to create it.
* Or by hand: `env\Scripts\activate`, then `python app.py`.
* A browser opens on `http://127.0.0.1:8080`. With `server.autostart_monitoring` (default) the scope is armed at launch.
* **Stop** with `Ctrl+C` in the command window (or close it). Monitoring can also be started/stopped from the screen.
* **Start at logon:** press `Win+R`, run `shell:startup`, and put a shortcut to `start_app.bat` there.
* **Without hardware:** `python app.py --simulate` (or `start_monitor.bat --simulate`).

A normal start prints, for example:
```
INFO controller: US Power Monitoring Controller 1.0.0 - driver 'auto' - http://127.0.0.1:8080/
INFO acquisition: Scope opened: {'model': 'PicoScope 2204A', 'serial': '...', 'driver': 'ps2000', ...}
INFO acquisition: Scope configured: {'dt': 4.096e-05, 'samples': 2441, 'window_s': 0.0999..., 'timebase': 12, ...}
```

## 6. Commissioning checklist

1. **Self-test** — `python app.py --selftest`. Opens the scope, captures once (auto-trigger on), and prints the timebase, the peak/RMS
   at each scope input, the scaled I and V, and warns if a channel clips. With no bond running it should show only noise (about 1 % of range).
2. **Ranges.** In the *Instrument* tab pick the smallest range that does not clip (peak roughly **50–80 % of range**; the 2204A is an 8-bit scope).
   A yellow **⚠ clipping** warning appears above the chart when a channel reaches full scale. Small signals on a large range look coarse and make the impedance noisy.
3. **Trigger.** Default is CH2 rising at 0.5 V **at the scope input** (= 5 V at the generator with a ×10 probe). Set it just above the noise
   so every bond starts at t = 0. *Auto-trigger* > 0 gives a free-running display for set-up; use 0 (wait) in production.
4. **Verify the scaling once** against a reference: a known load, or the same signals on your DSO. Check CT1 V/A, probe ratio, polarity
   and that the voltage, current and power peaks agree. Nothing here has been calibrated by software — the scale factors are the ones you enter.
5. Set **Power Limits** (W, per time range relative to the trigger), then **Save panel settings**.
6. Run a few real bonds and confirm alarms fire and are logged as expected (also try a deliberately out-of-limit bond).

## 7. Using the screen

**Header.** Status chips: ARMED / STOPPED, scope model (or fault), alarm state, clock. Under the Power chart title: the large **peak power** value.
Right side: last capture number, sample interval, points and time.

**Charts.**
* **Power (W)** — main chart: raw power (optional), positive-peak envelope (orange), max/min limit lines per time range (red), violation markers.
* **Voltage (V)**, **Current (A)** — positive (solid) and negative (dashed) peak envelopes, with large *peak* readouts.
* **Impedance (Ω)** — positive-peak V ÷ positive-peak I, axis always starts at 0 Ω, large *avg* readout.
* The chart holds the **last capture** until the next bond triggers.
* **Zoom / pan:** sliders, mouse wheel (zoom) and drag (pan) act on all four charts together; *Reset view* restores the full capture.
* **Envelope window (N)** — rolling window in samples (same as `plot_waveform_elvp.py`).
* **Z smooth (N)** — moving-average window for the impedance trace (default 51; 1 = off). **Z mask (%)** — impedance is blank where V or I is below this share of its peak (default 10 %).
* **Show raw traces** — overlay the unfiltered signals.

**Tabs (control panel).**
* **Power Limits** — grid of limit rows: *start*, *end* (seconds relative to trigger), *max* W, *min* W; add/delete rows (rows renumber automatically).
  *Negative power*: clamp to 0 or absolute value. **Power chart range**:
  *Smart auto* (default) holds the axis top on a 1-2-5 step, raises it at once when the power or a limit needs more room and lowers it only after 8 captures in a row that would fit under ~55 % of it, so the trace does not hop;
  *Fit every capture* rescales every time; *Fixed maximum* uses the typed value (W).
* **Instrument** (live mode) — CH1/CH2 range, coupling, CT1 V/A, probe ratio, invert; capture window (ms), samples per channel, pre-trigger %, trigger source/level/direction/auto-trigger. Changes apply to the running scope; the header shows the values the scope actually applied.
* **Waveform Simulation** (stand-alone simulator only) — base CSVs, random amplification per time range, update interval, point noise.
* **Save panel settings / Clear saved settings / Clear alarm log** — settings are stored **on the controller** (`data/panel_settings.json`) and reloaded automatically at start-up, for every browser.

**Alarm log.** *Power Failure Log* lists every alarm with time, range, HIGH/LOW, value and limit. The banner turns red while the current capture is out of limits.

## 8. Signal processing and the alarm rule

* **Scaling:** current = CH1 volts ÷ *volts_per_amp* (5 V/A for the CT1 into 50 Ω); voltage = CH2 volts × *probe_ratio*; either can be inverted.
* **Power:** per-sample V × I. Negative products are clamped to 0 (or made absolute, option *Negative power*) — "positive only".
* **Envelope:** rolling positive and negative peak over *N* samples (centered, minimum 1 period), identical to the pandas `rolling(window, center=True, min_periods=1)` used in `plot_waveform_elvp.py`.
* **Impedance:** positive-peak V of the smoothed envelope ÷ positive-peak I of the smoothed envelope, masked where either is below *Z mask* % of its peak. It is a peak ratio, not a phase-aware complex impedance.
* **Alarm rule:** for each limit row, only the **positive-peak power** inside its time window is examined. The highest value is compared with **Max** (→ *HIGH*) and the lowest with **Min** (→ *LOW*).
  One alarm per bound per row per capture. Changing a limit re-checks the current capture immediately. Negative-peak traces are never used for alarms.
* **Alarm record:** timestamp, capture number, limit row number, HIGH/LOW, time within the capture, power value, limit value.

## 9. Test signal from the scope generator (AWG)

The 2204A has an arbitrary waveform generator (4096 samples, 8-bit, ±2 V, 600 Ω source impedance). It can replay a DSO capture so the
whole chain (trigger, power calculation, limits, alarms) can be tested without a real transducer.

* **Inside the controller** (recommended): in `config.json` → `scope` set
  `"awg": {"enabled": true, "csv": "tek0001.csv", "scale": 1.0}` and restart. The log shows
  `AWG test signal started: tek0001.csv, 0.528 V p-p, repeats every 81.9 ms`.
  Wire the generator BNC to **CH1 and CH2** with a T-piece (into high impedance). Use smaller ranges (≈ ±0.5 V) and a trigger level around 0.1 V.
* **Stand-alone:** `python awg_play.py tek0001.csv` (add `--dry-run` to only show what would be played; `--scale` multiplies the amplitude).
  It cannot run together with `app.py` (the scope can only be opened by one program).
* **File handling:** a CSV of ≤ 4096 samples is played exactly as it is. A longer file has its active burst cut out and played at the original sample interval, padded with the baseline to 4096 samples (a burst longer than 4096 samples is decimated with a warning).
  Prepared 4096-sample versions of `tek0001.csv`: `tek0001_awg4096_burst.csv` (burst only, original 20 µs step, repeat 81.9 ms) and `tek0001_awg4096_full.csv` (whole 200 ms record stretched to 4096 points, 48.8 µs step).
* **Remember:** the DSO capture was taken at 50 kS/s, so any ultrasonic carrier in it was already under-sampled; the generator plays back those recorded samples, it does not recreate the real carrier. This tests the software, **not** the CT1/probe scaling.

## 10. Files written

| Path | Content |
|---|---|
| `logs/alarms_YYYY-MM-DD.csv` | one row per alarm: `timestamp, frame_seq, limit_range, type, time_s, power_w, limit_w` |
| `captures/capture_<stamp>_ALARM_seq<N>.csv` | the capture that alarmed, header `TIME,CH1,CH2` |
| `logs/controller.log` | rotating program log |
| `data/panel_settings.json` | limits and panel settings |
| `config.json` | instrument + server + logging settings (edited by the *Instrument* tab or by hand) |

Capture files are written so the existing scripts open them directly: **CH1 = amperes ÷ 0.1** (the scripts multiply CH1 by 0.1) and **CH2 = volts**.
`logging.save_captures`: `"alarms"` (default), `"all"`, or `"off"`; the oldest files are deleted beyond `max_capture_files`.

## 11. Settings reference (`config.json`)

| Key | Default | Meaning |
|---|---|---|
| `server.host` / `port` | `127.0.0.1` / `8080` | `127.0.0.1` = this PC only; `0.0.0.0` lets other PCs on the LAN view and **control** it (no login — keep it on a trusted network) |
| `server.open_browser` | `true` | open the browser at start |
| `server.autostart_monitoring` | `true` | arm the scope as soon as the program starts |
| `scope.driver` | `auto` | `auto`, `ps2000` (2204A/2205A), `ps2000a` (2205 MSO, 2206+), `sim`. Chosen at start-up only |
| `scope.ch1.range_v` / `coupling` | `2.0` / `AC` | input range (± V at the scope; 2204A: 0.02 – 20 V, 10 mV snaps to 20 mV) and `AC`/`DC` |
| `scope.ch1.volts_per_amp` | `5.0` | CT1 sensitivity: 5 mV/mA into 50 Ω |
| `scope.ch1.invert` | `false` | flip current polarity |
| `scope.ch2.range_v` / `coupling` | `5.0` / `AC` | as CH1 |
| `scope.ch2.probe_ratio` | `10.0` | voltage probe attenuation (10 = ×10) |
| `scope.ch2.invert` | `false` | flip voltage polarity |
| `scope.capture_window_ms` | `100.0` | time covered by one capture |
| `scope.samples` | `4000` | **upper limit** of samples per channel (8 kS memory shared by two channels). The 2204A timebase is a fixed table (powers of two), so the finest step that fits the window is used — see the applied values in the header (e.g. 40.96 µs × 2441 samples for 100 ms) |
| `scope.pre_trigger_pct` | `2.0` | share of the capture before the trigger point (time axis starts negative) |
| `scope.trigger.source` | `CH2` | `CH1`, `CH2` or `NONE` |
| `scope.trigger.level_v` | `0.5` | volts **at the scope input** (after the probe) |
| `scope.trigger.direction` | `RISING` | `RISING` / `FALLING` |
| `scope.trigger.auto_ms` | `0` | 0 = wait for a trigger; > 0 = free-run after this many ms (set-up only) |
| `scope.awg` | disabled | test signal: `{"enabled": false, "csv": "tek0001.csv", "scale": 1.0}` (section 9), applied at start-up |
| `logging.directory` / `captures_directory` | `logs` / `captures` | output folders |
| `logging.save_captures` | `alarms` | `off`, `alarms`, `all` |
| `logging.max_capture_files` | `2000` | oldest capture files are deleted beyond this |

Operator panel settings (limits, envelope window, Z smooth/mask, power chart range, negative-power mode) are **not** in `config.json`; they are saved with *Save panel settings* in `data/panel_settings.json`.

## 12. Command-line options and HTTP API

```
python app.py [--config PATH] [--simulate] [--selftest] [--diagnose] [--host HOST] [--port PORT] [--no-browser]
```

| Option | Effect |
|---|---|
| `--simulate` | use the built-in simulated scope |
| `--selftest` | capture once and print a commissioning report |
| `--diagnose` | find out why the PicoScope cannot be opened (section 16) |
| `--host` / `--port` | override `server.host` / `server.port` |
| `--no-browser` | do not open the browser automatically |
| `--config` | use another `config.json` |

**HTTP / WebSocket** (used by the screen; also usable by other software on the same network):

| Endpoint | Purpose |
|---|---|
| `GET /` | the monitor screen |
| `GET /ws` | WebSocket: status messages (JSON) and binary frames (`u32` header length, JSON header, `float32` current[], `float32` voltage[]) |
| `GET /api/info` | application name/version |
| `GET` / `PUT /api/scope` | read / change instrument settings (validated, applied to the running scope) |
| `POST /api/acquire` | start / stop monitoring |
| `GET` / `PUT` / `DELETE /api/panel` | read / save / delete the stored panel settings |
| `POST /api/alarm` | the screen reports an alarm for logging and capture saving |

State-changing calls require a JSON content type and an allowed browser origin. There is **no login**.

## 13. Limits of this design

* **Sampling.** The 2204A has 8 kS of memory shared by both channels. A 100 ms window is therefore about **24 kS/s (Nyquist ≈ 12 kHz)** — an ultrasonic carrier of tens of kHz is **under-sampled**. This is the same situation as the original 1000-point DSO captures: V and I are still sampled at the *same* instants, and the rolling positive-peak envelope recovers an amplitude, but the result depends on sampling phase and the instantaneous product is not a true V·I waveform. A shorter window raises the sample rate but covers less of the bond. For exact power over the whole bond, a streaming-mode driver is the next step (section 14).
* **Resolution.** The scope is 8-bit. Signals using a small part of the input range are quantised coarsely, which makes impedance (a ratio) noisy. Choose ranges so peaks reach 50–80 % of full scale.
* **"Power" definition.** Following the requirement, power is the per-sample product V × I kept positive, and limits apply to its positive-peak envelope. This is a peak figure, **not average (active) power**.
* **CT1** has no DC response, droops below ~25 kHz, and can saturate (1 A·s). Impedance is a peak ratio, not a complex impedance.
* **No calibration.** Scale factors are the numbers you enter; verify them against a reference.
* **Single user of the scope.** Only one program can open the scope at a time.
* **Security.** The web interface has no login. Keep `host` at `127.0.0.1`, or use it only on a trusted network.
* **Not a safety interlock.** Alarms are an indication and a log. Do not make the browser or this PC the only thing that stops a bad bond.
* **Unverified on hardware:** trigger delay (pre-trigger) behaviour and trigger direction codes of the `ps2000` API follow Pico's documentation; confirm with a real signal.

## 14. Extending the application

* **Alarm output** (tower light, buzzer, reject signal): add it in `Controller.log_alarm` (`app.py`) — e.g. a USB relay or a Modbus/OPC-UA write — or drive it from the PLC by reading `logs/alarms_*.csv`.
* **Exact power over the full bond:** add a streaming-mode driver beside `scope_pico2000.py` (`ps2000_run_streaming_ns`, ~µs interval, hardware trigger, compute V×I at full rate in numpy and send peak-reduced series). `acquisition.py` and the UI need no change as long as the frame keeps the same fields.
* **Other scopes / DAQ:** implement the methods used by `acquisition.py` (`open, configure, arm, is_ready, read, stop, close`) and return a `ScopeRaw`; register it in `make_scope` (`app.py`) and the `driver` choices (`config.py`).
* **Rolling-mean (active) power:** add an option beside *Negative power* in `web/index.html` (`powerOf`, `recomputeDerived`).
* **Authentication / remote access:** put the app behind a reverse proxy with login before exposing it beyond a trusted LAN.

## 15. Tests

```
python tests/test_pico2000_fake.py      # 2204A driver against a fake picosdk.ps2000 (7 tests)
python tests/test_pico_driver_fake.py   # ps2000a driver against a fake picosdk.ps2000a (6 tests)
```
Set `PICOSDK_SRC` to a checkout of `picosdk-python-wrappers` to also verify that every driver function the code calls exists in the real wrapper.
The fake-driver tests prove our call sequence, timebase search, trigger conversion, buffer handling and scaling; they cannot prove the real DLL behaves like the fake.
The whole application can be exercised without hardware with `python app.py --simulate`.

## 16. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `No PicoScope found` / `PICO_NOT_FOUND` | (1) **PicoScope 6/7 is running** and holds the scope — close it fully, also in the tray. (2) A 2204A is not a `ps2000a` device — use `driver: "auto"` or `"ps2000"` (default). (3) USB cable / port. Run **`python app.py --diagnose`**: it lists running Pico programs, what Windows sees on USB, what each driver enumerates, and tries to open the scope with each driver |
| **SCOPE FAULT: driver could not be loaded** | install the 64-bit PicoSDK / PicoScope 7; use 64-bit Python; `pip install picosdk` |
| `The PicoScope reports no valid timebase` | send the full message; it lists the rejected timebases |
| Stays **ARMED – waiting for trigger** | trigger level too high for the signal (level is volts at the scope input); try *Auto-trigger* 500 ms to see the signal, then set the level just above the noise |
| ⚠ clipping | choose a larger range for that channel |
| Signal tiny / impedance very noisy | range too large for the signal; choose a smaller range (50–80 % of full scale); raise *Z smooth* |
| Power reads ~half / double | CT1 not terminated in 50 Ω, or wrong *volts per amp*; wrong probe ratio |
| Power always 0 | current and voltage opposite polarity — tick *Invert CH1* (or CH2) |
| Power chart axis hops | set *Power chart range* to *Smart auto* or *Fixed maximum* |
| `awg.csv not found` / generator rejected | put the CSV next to `app.py` (or use a full path); the message in the banner/log names the cause |
| Browser cannot connect | the program is not running; check the port in `config.json`; try `http://127.0.0.1:8080` |
| `pip` / package errors | Python must be 3.10+ and 64-bit; run `pip install -r requirements.txt` inside the activated `env` |

For any fault, the program log `logs/controller.log` and the command window show the exact message; include it when asking for help.

## 17. Sources

* [Tektronix CT-1 / CT-2 datasheet](https://xdevs.com/doc/Tektronix/CT-1_CT-2.pdf)
* [PicoScope 2204A specifications](https://www.tme.eu/details/picoscope2204a) and [signal-generator / AWG specifications](https://interworldna.com/pico/2200a_specifications.php)
* [Pico Technology picosdk-python-wrappers](https://github.com/picotech/picosdk-python-wrappers) (`ps2000` and `ps2000a` examples)
* PicoScope 2000 Series Programmer's Guide (`ps2000` API) and PicoScope 2000A Series Programmer's Guide (`ps2000a` API), Pico Technology
