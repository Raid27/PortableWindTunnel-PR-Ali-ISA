# PortableWindTunnel-PR-Ali-ISA

# 🌬️ Portable Wind Tunnel

A portable, low-cost wind tunnel built on an Arduino Uno R3 with bare-metal AVR firmware, a Python data pipeline, live dashboard, PID fan control, a simple CFD simulation, and an ML flow classifier. The project is designed to be a hands-on end-to-end embedded + data science stack built from scratch.

---

## Table of contents

- [Overview](#overview)
- [Building the tunnel — mechanical design](#building-the-tunnel--mechanical-design)
  - [Design principles](#design-principles)
  - [Test section](#test-section)
  - [Flow straightener](#flow-straightener)
  - [Fan mounting](#fan-mounting)
  - [Pitot tube fabrication](#pitot-tube-fabrication)
  - [Sealing and checking for leaks](#sealing-and-checking-for-leaks)
  - [Test objects](#test-objects)
- [Hardware](#hardware)
  - [Parts list](#parts-list)
  - [Wiring](#wiring)
- [Repository structure](#repository-structure)
- [Phase 1 — Firmware (AVR bare-metal C)](#phase-1--firmware-avr-bare-metal-c)
- [Phase 2 — Python data pipeline](#phase-2--python-data-pipeline)
- [Phase 3 — Live dashboard](#phase-3--live-dashboard)
- [Phase 4 — PID fan control](#phase-4--pid-fan-control)
- [Phase 5 — CFD simulation](#phase-5--cfd-simulation)
- [Phase 6 — ML flow classifier](#phase-6--ml-flow-classifier)
- [Setup](#setup)
- [Usage](#usage)
- [Physics reference](#physics-reference)
- [Contributing](#contributing)

---

## Overview

```
Fan → Flow straightener → Test section (Pitot tube + pressure sensor) → Outlet
         ↑                         ↓
    PWM via AVR Timer1        ADC → UART → Python → Dashboard / ML
```

The tunnel produces a controllable laminar airflow over a test section. A Pitot tube measures dynamic pressure, which is converted to velocity via the Bernoulli equation. Everything downstream — logging, visualization, control, and machine learning — runs in Python on a PC connected over USB serial.

**Stack at a glance**

| Layer | Technology |
|---|---|
| MCU | Arduino Uno R3 (ATmega328P), bare-metal C via PlatformIO |
| Sensor | MPXV7002DP differential pressure sensor |
| Fan control | Timer1 fast PWM, 25 kHz, IRLZ44N gate driver |
| Communication | UART at 9600 baud, CSV format |
| Data pipeline | Python + pyserial + pandas |
| Dashboard | Plotly Dash or matplotlib FuncAnimation |
| Control | PID controller implemented in AVR C |
| Simulation | 2D finite-difference CFD in numpy |
| ML | scikit-learn classifier on time-series features |

---

## Building the tunnel — mechanical design

Before any firmware or code, the physical tunnel needs to be built well. Flow quality in the test section is entirely a function of how well the tunnel is constructed — a leaky or misaligned tunnel gives noisy, unusable data regardless of how good the firmware is.

### Design principles

A low-speed open-circuit tunnel has four functional zones:

```
[Inlet bell]──[Flow straightener]──[Contraction]──[Test section]──[Diffuser]──[Fan]──[Exhaust]
```

For a portable benchtop version, the contraction and diffuser can be simplified or omitted, but the flow straightener is non-negotiable.

### Test section

The test section is the heart of the tunnel — it needs to be rigid, sealed, and optically accessible if you want to use smoke visualization.

**Dimensions:** aim for a square cross-section of at least 15 × 15 cm and a length of 25–35 cm. Longer is better — the flow needs space to become uniform after the straightener.

**Material options:**

| Material | Pros | Cons |
|---|---|---|
| 6 mm acrylic sheet | Transparent (smoke vis), rigid, easy to cut | Brittle, cracks if over-drilled |
| 12 mm plywood | Cheap, easy to work, stiff | Opaque, seams need sealing |
| 3D-printed PLA panels | Precise geometry, easy iteration | Print seams leak, warps under heat |

Acrylic is recommended if you want smoke visualization. Use acrylic cement (not hot glue) at seams — hot glue shrinks and leaks. Sand all interior surfaces smooth; surface roughness adds turbulence.

**Construction steps:**

1. Cut four side panels to identical dimensions using a table saw or laser cutter
2. Join panels at 90° using acrylic cement or wood glue + corner brackets; check square with a try-square before the adhesive sets
3. Leave both ends fully open (inlet and outlet)
4. Drill a Pitot tube port at mid-length, mid-height: a tight 4 mm hole with a rubber grommet to seal around the tube shaft
5. Optional: cut a 5 × 5 cm access port in one side panel with a sliding cover, for placing test objects without disassembling the tunnel

### Flow straightener

Without a straightener, the fan generates a rotating, turbulent inflow that ruins measurements. A honeycomb straightener breaks up the swirl into parallel streams.

**Construction:**

1. Cut a bundle of drinking straws (or PVC tubes, 4–6 mm ID) to a length of 5–8 cm — longer cells give better straightening
2. Pack them tightly into a frame that friction-fits into the tunnel inlet cross-section
3. Zip-tie or tape the bundle on the outside to hold the pack together
4. The cell length-to-diameter ratio should be at least 6:1 for effective straightening — standard drinking straws at 6 mm ID and 6 cm cut length give exactly 10:1, which is ideal

> **Why this matters:** Without a straightener, measured velocity can vary by ±30% across the test section. With a well-made honeycomb, uniformity within ±5% is achievable on a benchtop build.

### Fan mounting

The fan mounts at the downstream end (blowing out, not in) — this is a **draw-down** configuration. It pulls air through the test section rather than pushing it, which keeps the fan turbulence downstream of your measurements.

**Steps:**

1. Cut a fan mounting plate from 6 mm plywood or acrylic matching your tunnel's cross-section
2. Cut a circular hole slightly smaller than the fan frame, or use the fan's corner mounting holes
3. Mount the fan with M3 screws and rubber grommets between the fan frame and plate — this decouples fan vibration from the tunnel structure, which otherwise shows up as low-frequency noise in your pressure readings
4. Seal around the fan plate edges with foam weatherstripping tape

**Fan placement note:** leave a gap of at least one tunnel width (15–20 cm) between the test section outlet and the fan inlet. This prevents the fan's pressure field from affecting the test section.

### Pitot tube fabrication

A Pitot-static tube has two concentric pressure ports: the stagnation (total pressure) port at the nose, and static pressure ports around the shaft.

**Simple version using brass tube:**

1. Take a 30 cm length of 3 mm OD brass tube (total pressure line)
2. Slide it inside a 6 mm OD brass tube (static pressure line), centered
3. Solder the inner tube closed at the nose end, leaving just the tip open facing into the flow
4. Drill 4–6 × 0.5 mm static ports radially around the 6 mm outer tube, ~8 tube-diameters back from the nose
5. Seal the annular gap between inner and outer tubes at the rear with epoxy, bringing out two separate silicone tubes: one from the inner (total) and one from the outer (static)
6. Connect these two silicone tubes to the P1 and P2 ports on the MPXV7002DP

**Alignment:** mount the Pitot tube so the nose faces directly into the flow (0° yaw). A ±5° misalignment introduces a cosine error of ~0.4%, which is acceptable. Beyond ±15°, error grows rapidly.

### Sealing and checking for leaks

Leaks are the most common source of measurement error in a homebuild tunnel. Before any electrical work:

1. Block the outlet with your hand and use a hair dryer at low speed into the inlet — feel for air escaping from seams
2. Seal any leaks with silicone RTV sealant, not hot glue
3. Check the Pitot port grommet: it should be snug enough that there's no bypass airflow around the tube shaft

### Test objects

The tunnel becomes much more interesting with things to put in it. Some easy starting objects:

| Object | What you learn |
|---|---|
| Flat plate at various angles | Drag vs angle of attack, stall |
| Cylinder (a pen or dowel) | Bluff body drag, Kármán vortex shedding |
| Symmetric airfoil (3D-printed NACA 0012) | Lift/drag ratio, compare to CFD |
| Car body model | Real-world aero comparison |

Mount test objects on a thin rod through a sealed port in the floor of the test section. If you add a small strain gauge or load cell to the mount rod, you can measure drag force directly — a compelling extension to the project.

---

## Hardware

### Parts list

| Component | Part / spec | Notes |
|---|---|---|
| Fan | 120 mm 4-pin PWM brushless DC, 12 V | 4-pin preferred for hardware PWM tach |
| Pressure sensor | MPXV7002DP (±2 kPa differential) | Analog output, directly to AVR ADC0 |
| Gate driver / MOSFET | IRLZ44N + 10 Ω gate resistor | Or L298N if you already have one |
| Flyback diode | 1N4007 across fan terminals | Required to protect the MOSFET |
| MCU | Arduino Uno R3 (ATmega328P) | Any 5 V AVR will work |
| Test section box | ~20 × 20 × 30 cm acrylic or plywood | Open inlet and outlet ends |
| Flow straightener | Bundle of drinking straws at inlet | ~5 cm deep, zip-tied together |
| Pitot tube | Brass or 3D-printed, ~3 mm OD | Static port and dynamic port |
| Button | Normally-open tactile switch | Fan speed step control |

### Wiring

```
AVR Pin 9  (OC1A)  ──[10 Ω]──> IRLZ44N Gate
                                IRLZ44N Drain ──> Fan –
                                IRLZ44N Source ──> GND
                                1N4007 across Fan+ / Fan–

AVR Pin A0 (ADC0)  ──────────> MPXV7002DP VOUT
                                MPXV7002DP VCC  ──> +5 V
                                MPXV7002DP GND  ──> GND
                    Pitot dynamic port ──> MPXV7002DP P1
                    Pitot static port  ──> MPXV7002DP P2

AVR PD2 (Pin 2)    ──[10 kΩ pullup]──> Button ──> GND

AVR TX (Pin 1)     ──> USB-Serial (via Uno onboard CH340/ATmega16U2)
```

---

## Repository structure

```
wind-tunnel/
├── firmware/               ← PlatformIO project (AVR bare-metal C)
│   ├── platformio.ini
│   └── src/
│       └── main.c
├── python/
│   ├── serial_logger.py    ← reads UART, writes timestamped CSV
│   ├── dashboard.py        ← live Plotly Dash visualization
│   ├── pid_tuner.py        ← step-response logger for PID tuning
│   ├── cfd_sim.py          ← 2D finite-difference CFD solver
│   └── ml/
│       ├── collect_data.py ← label + record dataset windows
│       ├── train.py        ← feature extraction + model training
│       └── inference.py    ← live inference on serial stream
├── data/                   ← session CSV logs (gitignored)
├── docs/
│   └── wiring.png
└── README.md
```

---

## Phase 1 — Firmware (AVR bare-metal C)

All firmware lives in `firmware/src/main.c`. No Arduino library functions — raw AVR registers only.

### Key register configuration

**ADC** — single-ended read on ADC0 (pressure sensor):

```c
ADMUX  = (1 << REFS0);                  // AVcc reference, ADC0
ADCSRA = (1 << ADEN) | (1 << ADPS2)
       | (1 << ADPS1) | (1 << ADPS0);  // enable, prescaler /128
```

**Timer1 fast PWM** — 25 kHz on OC1A (Pin 9) for fan:

```c
ICR1   = 639;          // TOP = F_CPU / (prescaler * freq) - 1 = 16MHz/1/25kHz - 1
TCCR1A = (1 << COM1A1) | (1 << WGM11);
TCCR1B = (1 << WGM13) | (1 << WGM12) | (1 << CS10);  // no prescaler
OCR1A  = 0;            // duty: 0 = off, ICR1 = 100%
DDRD  |= (1 << PD1);   // OC1A as output
```

**UART TX** — 9600 baud:

```c
#define BAUD 9600
#define UBRR_VAL (F_CPU / 16 / BAUD - 1)   // = 103
UBRR0H = (UBRR_VAL >> 8);
UBRR0L =  UBRR_VAL;
UCSR0B = (1 << TXEN0);
UCSR0C = (1 << UCSZ01) | (1 << UCSZ00);    // 8N1
```

### Serial output format

Each line sent over UART:

```
velocity_ms,raw_adc,fan_duty_pct\n
```

Example: `3.42,487,60\n`

### Velocity calculation

Using Bernoulli's equation for incompressible flow:

```
v = sqrt(2 * ΔP / ρ)
```

Where `ΔP` (Pa) is derived from the MPXV7002DP transfer function:

```
ΔP = ((Vout / Vcc) - 0.5) * 4000    [Pa, range ±2000 Pa]
```

And `ρ = 1.225 kg/m³` (air at sea level, 15 °C).

---

## Phase 2 — Python data pipeline

```bash
pip install pyserial pandas
```

`serial_logger.py` opens the COM port, parses each CSV line, and appends rows to a dated session file:

```python
import serial, csv, time, datetime

PORT = "/dev/ttyUSB0"   # or "COM3" on Windows
BAUD = 9600

with serial.Serial(PORT, BAUD, timeout=1) as ser, \
     open(f"data/session_{datetime.date.today()}.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["timestamp", "velocity", "raw_adc", "fan_duty"])
    while True:
        line = ser.readline().decode("ascii", errors="ignore").strip()
        if not line:
            continue
        parts = line.split(",")
        if len(parts) == 3:
            writer.writerow([time.time()] + parts)
```

---

## Phase 3 — Live dashboard

```bash
pip install plotly dash
python python/dashboard.py
```

Opens a browser at `http://localhost:8050` with a live velocity time-series plot updating at ~10 Hz. The fan duty slider sends a single byte back to the AVR over serial to adjust speed.

---

## Phase 4 — PID fan control

A discrete PID controller runs in `main.c` on the AVR, called inside the main loop at a fixed ~100 ms interval. The setpoint can be sent over UART as a single ASCII byte (target velocity in m/s).

```c
float pid_update(float setpoint, float measured, float dt) {
    float error    = setpoint - measured;
    integral      += error * dt;
    integral       = clamp(integral, -MAX_I, MAX_I);   // anti-windup
    float deriv    = (prev_measured - measured) / dt;  // on measurement
    prev_measured  = measured;
    return Kp * error + Ki * integral + Kd * deriv;
}
```

Tune `Kp`, `Ki`, `Kd` by running `python/pid_tuner.py`, which logs a step response and plots settling time and overshoot.

**Starting values (tune from here):**

| Gain | Initial value |
|---|---|
| Kp | 50.0 |
| Ki | 5.0 |
| Kd | 1.0 |

---

## Phase 5 — CFD simulation

`python/cfd_sim.py` implements a simple 2D incompressible flow solver using finite differences and a pressure-correction (SIMPLE-like) scheme.

```bash
pip install numpy matplotlib scipy
python python/cfd_sim.py
```

The solver produces a velocity field `u[i,j], v[i,j]` and pressure field `p[i,j]` on a structured grid. A flat plate or cylinder can be placed in the test section by masking grid cells and enforcing no-slip on the object surface.

**Boundary conditions:**

| Boundary | Condition |
|---|---|
| Inlet (left) | `u = U_inf`, `v = 0` |
| Outlet (right) | Zero-gradient (Neumann) |
| Top / bottom walls | No-slip: `u = v = 0` |
| Object surface | No-slip |

After solving, the script overlays the simulated pressure at the Pitot tube's grid location against the real measured `ΔP` from your CSV logs.

---

## Phase 6 — ML flow classifier

### Data collection

```bash
python python/ml/collect_data.py --label laminar --duration 60
python python/ml/collect_data.py --label turbulent --duration 60
```

Records labeled windows of sensor data. Aim for ≥ 200 windows per class. Laminar = low fan speed with no obstructions. Turbulent = high fan speed or an obstruction placed in the test section.

### Features extracted per window

| Feature | Description |
|---|---|
| `mean` | Mean velocity over window |
| `std` | Standard deviation |
| `peak_to_peak` | Max − min velocity |
| `fft_peak_freq` | Dominant FFT frequency |
| `zero_crossing_rate` | Rate of sign changes in velocity fluctuation |

### Training

```bash
python python/ml/train.py
```

Trains a `RandomForestClassifier` (baseline) and prints accuracy, confusion matrix, and learning curves. Target: ≥ 85% validation accuracy.

### Live inference

```bash
python python/ml/inference.py
```

Buffers the last N samples from the serial stream, extracts features, calls `model.predict()`, and displays the current flow state in the terminal (and on the dashboard if running).

---

## Setup

### Firmware

1. Install [PlatformIO](https://platformio.org/) in VS Code.
2. Open `firmware/` as a PlatformIO project.
3. `platformio.ini` should specify `board = uno`, `framework = arduino` overridden to bare-metal (no Arduino library includes in `main.c`).
4. Build and upload: `pio run --target upload`.

### Python environment

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install pyserial pandas plotly dash numpy matplotlib scipy scikit-learn
```

### Running the full stack

```bash
# Terminal 1 — log data
python python/serial_logger.py

# Terminal 2 — dashboard
python python/dashboard.py

# Terminal 3 (optional) — live ML inference
python python/ml/inference.py
```

---

## Usage

| Action | How |
|---|---|
| Start fan at fixed speed | Set `OCR1A` in firmware or use dashboard slider |
| Set PID target velocity | Send target m/s as ASCII byte over serial |
| Log a session | Run `serial_logger.py` before starting the fan |
| Collect ML training data | Run `collect_data.py` with `--label` flag |
| Run CFD comparison | Run `cfd_sim.py` after collecting a session CSV |

---

## Physics reference

| Symbol | Meaning | Value / formula |
|---|---|---|
| ρ | Air density | 1.225 kg/m³ (sea level, 15 °C) |
| ΔP | Dynamic pressure from Pitot | `P_total − P_static` |
| v | Flow velocity | `sqrt(2ΔP / ρ)` |
| Re | Reynolds number | `ρvL / μ` — determines laminar vs turbulent |
| Cd | Drag coefficient | `F_drag / (0.5 * ρ * v² * A)` |
| μ | Dynamic viscosity of air | 1.81 × 10⁻⁵ Pa·s |

**MPXV7002DP transfer function:**

```
Vout = Vcc * (0.5 + 0.2 * (P1 - P2) / kPa_max)
     → ΔP [Pa] = ((Vout / Vcc) - 0.5) * 4000
```

---

## Contributing

1. Fork the repo and create a feature branch.
2. Firmware changes: keep everything in bare-metal AVR C — no `#include <Arduino.h>`.
3. Python changes: add tests in `python/tests/` where practical.
4. Open a PR with a brief description of the change and any relevant data or plots.

---

*Built with an Arduino Uno R3, a box, some straws, and a lot of Bernoulli.*
