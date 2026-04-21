# PortableWindTunnel-PR-Ali-ISA

# 🌬️ Portable Wind Tunnel

A portable, low-cost wind tunnel built on an Arduino Uno R3 with bare-metal AVR firmware, a Python data pipeline, live dashboard, PID fan control, a simple CFD simulation, and an ML flow classifier. The project is designed to be a hands-on end-to-end embedded + data science stack built from scratch.

---

## Table of contents

- [Overview](#overview)
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
