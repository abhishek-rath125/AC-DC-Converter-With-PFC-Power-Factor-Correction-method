# AC-DC Boost Converter with Cascaded Control in Simulink

A step-by-step Simulink project that builds an AC to DC boost stage from a bare rectifier up to a closed-loop converter that regulates a **48 V / 16 A** load from a **30 V rms, 50 Hz** supply.

The project is organised in four stages. Each stage lives in its own folder with its model, screenshots and an explanation.

| Stage | Folder | What it adds | Output voltage | Load current |
|---|---|---|---|---|
| 1 | [`1.Rectifier`](01-rectifier) | Diode bridge rectifier with a resistive load | Peak about 40.8 V (rectified, no regulation) | not measured |
| 2 | [`2.ACDC Boost Converter`](02-ac-dc-boost-converter) | Boost converter after the rectifier, open loop with fixed duty cycle 0.5 | 47.53 V | 15.84 A |
| 3 | [`3.Cascaded PI Control with Inner and Outer Current and Voltage loop](03-cascaded-pi-control) | Outer PI voltage loop + inner PI current loop + PWM | 47.8 V | 15.93 A |
| 4 | [`4.SR FlipFlop Control`](04-sr-flipflop-control) | Outer PI voltage loop + current comparator + S-R flip-flop (peak current mode) | 48.01 V | 16 A |

Target: 48 V and 16 A (768 W, 3 ohm load).

---

## Contents
- [Project idea](#project-idea)
- [Common specification](#common-specification)
- [Stage overview](#stage-overview)
- [Repository structure](#repository-structure)
- [How to run](#how-to-run)
- [Lessons learned](#lessons-learned)
- [Roadmap](#roadmap)
- [Documentation](#documentation)
- [Author and license](#author-and-license)

---

## Project idea

A diode bridge with a large capacitor draws short current pulses near the voltage peaks. That gives a poor power factor and many harmonics. In this project there is **no capacitor right after the rectifier**. A boost converter sits directly behind it, and its controller decides the shape of the input current. The long-term goal is a converter whose input current follows the rectified supply voltage (unity power factor behaviour), so that the source sees something close to a resistor.

The theory behind this (why a boost converter, why unity power factor, outer and inner loops, peak current mode) is in [`docs/theory.md`](docs/theory.md).

## Common specification

| Item | Value |
|---|---|
| Source | 30 V rms, 50 Hz (block set to `30*sqrt(2)` V peak) |
| Output target | 48 V, 16 A (768 W), load about 3 ohm |
| Inductor (stages 3 and 4) | 1 mH |
| Output capacitor (stages 3 and 4) | 120 mF |
| Switching frequency (stages 3 and 4) | 20 kHz (period 5e-5 s) |
| Solver, stages 1 and 2 | powergui Continuous |
| Solver, stages 3 and 4 | powergui Discrete, fixed step 1e-6 s |

## Stage overview

### Stage 1: Rectifier
A single-phase diode bridge feeds a resistor. The scope compares the source voltage with the rectified voltage. This confirms the supply (peak about 42.4 V) and shows that the bridge output is a 100 Hz rectified sine that never exceeds the source peak.

![Rectifier output](01-rectifier/images/rectifier_output.png)

### Stage 2: Open-loop boost converter
A MOSFET, inductor, diode and output capacitor are added after the bridge. A constant 0.5 drives the PWM Generator, so the duty cycle is fixed. The output reaches about 47.5 V, but nothing regulates it: any change in load, supply or duty cycle moves it.

![Open-loop boost output](02-ac-dc-boost-converter/images/boost_open_loop_output.png)

### Stage 3: Cascaded PI control (outer voltage loop and inner current loop)
The duty cycle is no longer fixed. The voltage PI output becomes the inductor current reference, and a current PI turns the current error into a duty cycle for the PWM Generator.

![Cascaded PI output](03-cascaded-pi-control/images/cascaded_pi_output.png)

### Stage 4: S-R flip-flop (peak current mode)
The current PI and PWM Generator are replaced by a comparator and an S-R flip-flop. A clock turns the MOSFET on, and the comparator turns it off when the inductor current reaches the reference. The reference is the voltage PI output multiplied by the normalised rectified voltage, so it follows the shape of the supply.

![S-R flip-flop output](04-sr-flipflop-control/images/srff_output.png)

> **Note:** these results show **output regulation**. Input power factor and harmonic distortion have not been measured yet; see the [Roadmap](#roadmap).

## Repository structure

```
.
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── theory.md                    # Background and control theory
│   └── troubleshooting.md           # Errors met during the project and their fixes
├── 01-rectifier/
│   ├── README.md
│   ├── PE.slx                       # (add your model here)
│   └── images/
├── 02-ac-dc-boost-converter/
│   ├── README.md
│   ├── PE_Boost.slx                 # (add your model here)
│   └── images/
├── 03-cascaded-pi-control/
│   ├── README.md
│   ├── PE_version1.slx              # (add your model here)
│   └── images/
└── 04-sr-flipflop-control/
    ├── README.md
    ├── PE_version2.slx              # (add your model here)
    └── images/
```

## How to run

**Requirements:** MATLAB with Simulink and Simscape Electrical (the models use the `powergui` block and the Specialized Power Systems library). Add your MATLAB release here: `R20xx`.

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

Then open the stage you want in MATLAB:

```matlab
cd 01-rectifier;               open_system('PE')
cd ../02-ac-dc-boost-converter; open_system('PE_Boost')
cd ../03-cascaded-pi-control;   open_system('PE_version1')
cd ../04-sr-flipflop-control;   open_system('PE_version2')
```

Stages 3 and 4 need the powergui set to **Discrete** with sample time `1e-6` and a fixed-step solver with step `1e-6`.

## Lessons learned

- **Zero-crossing chatter** between the comparator and the power-stage solver stopped the stage 4 simulation. Fix: discrete powergui and a fixed-step solver.
- **Continuous states** with a fixed-step discrete solver: the PI blocks must be discrete (`PI(z)`).
- **Pulse Generator sample time:** period and width must be integer multiples of the fixed step.
- **Output stuck low while the voltage PI sat at its limit:** check that the normalised voltage peaks at 1, that source amplitude and gain agree (peak vs rms), and that the PI limit is realistic for 768 W.

Full list: [`docs/troubleshooting.md`](docs/troubleshooting.md).

## Roadmap

- [ ] Measure input power factor and THD in stages 3 and 4 (powergui FFT on the source current)
- [ ] Plot source voltage and source current together to confirm they are in phase
- [ ] Add slope compensation to the stage 4 peak current loop
- [ ] Add a soft start (ramp on the 48 V reference)
- [ ] Load step and line step tests
- [ ] Reduce the 100 Hz ripple in the current reference (slower voltage loop or filtered feedback)
- [ ] Efficiency estimate with realistic device losses

## Documentation

| File | Content |
|---|---|
| [`docs/theory.md`](docs/theory.md) | Why a boost converter, unity power factor, outer and inner loops, peak current mode |
| [`docs/troubleshooting.md`](docs/troubleshooting.md) | Errors met and how they were solved |
| [`01-rectifier/README.md`](01-rectifier/README.md) | Stage 1 |
| [`02-ac-dc-boost-converter/README.md`](02-ac-dc-boost-converter/README.md) | Stage 2 |
| [`03-cascaded-pi-control/README.md`](03-cascaded-pi-control/README.md) | Stage 3 |
| [`04-sr-flipflop-control/README.md`](04-sr-flipflop-control/README.md) | Stage 4 |

## Author and license

Created by **Abhishek**, B.Tech EEE, NIT Goa, as part of power electronics coursework.

Released under the [MIT License](LICENSE).
