# Theory and design background

## 1. Why a boost converter?

The source is 30 V rms at 50 Hz. After the diode bridge the voltage is a rectified sine with a peak of 30 x sqrt(2) = 42.4 V, and it drops to 0 V twice per line cycle. The required output is 48 V.
The input is always below the output, so a step-up (boost) converter is needed.

## 2. Why no capacitor after the rectifier?

With a large capacitor after the bridge, current flows only near the voltage peaks (short, tall pulses). This gives:
- poor power factor and a high RMS current for the same real power,
- strong low-order harmonics (3rd, 5th, 7th, ...),
- higher stress on the source and wiring.

Without that capacitor, the boost converter can shape the input current. If the current is a rectified sine **in phase** with the rectified voltage, the bridge plus converter behaves like a **resistor** as seen by the source:
- current in phase with voltage (power factor close to 1),
- low harmonic content,
- only real power is drawn.

This is the idea of **unity power factor (UPF)** operation. "Current stiff" in the lecture notes means the converter forces the current waveform to be the same as the rectified voltage waveform.

## 3. Power pulsation and the output capacitor

With a sinusoidal input voltage and current in phase, the instantaneous input power is proportional to sin^2(wt), which pulsates at **twice the line frequency (100 Hz)**. The load needs constant power (48 V x 16 A = 768 W). The difference is stored in and released from the output capacitor (E = 1/2 C V^2), so the output voltage carries a small 100 Hz ripple. This is visible on the output plots.

## 4. Cascaded control: outer voltage loop and inner current loop

**Why the voltage controller outputs a current reference.** The output voltage is changed by moving charge in the output capacitor:

C dVo/dt = i_diode - i_load

So the only way to change Vo is to change the current delivered through the inductor. The voltage loop therefore asks: *how much current do I need?* Its output is a current reference, not a voltage.

| Loop | Speed | Job |
|---|---|---|
| Outer (voltage) | Slow (well below 100 Hz) | Compare Vo with 48 V, set the **amplitude** of the current reference |
| Inner (current) | Fast (switching frequency) | Force the inductor current to follow the reference, which also sets the **shape** |

Reasons for two loops:
- Inductor current responds quickly, output voltage slowly, so the loops can be tuned separately.
- The current reference can be limited, which protects the inductor and MOSFET.
- A boost converter has a right-half-plane zero in its duty-to-voltage response, which makes direct voltage control hard. Controlling the inductor current first avoids most of that difficulty.
- Input voltage disturbances are corrected by the inner loop before they reach the output.

The outer loop must stay slow. If it reacted to the 100 Hz ripple on Vo, that ripple would pass into the current reference and distort the input current.

## 5. Shaping the current for unity power factor

For UPF, the current reference must have the shape of the rectified voltage:

i_ref(t) = I_amp x |v_in(t)| / V_peak

- `I_amp` comes from the voltage PI.
- `|v_in(t)| / V_peak` is the measured rectified voltage divided by its peak (signal `[E]`, peak = 1). Measuring the actual voltage avoids a phase mismatch that a separate sine generator could have.

Stage 3 (cascaded PI control) uses the voltage PI output directly as the current reference (no multiplication), so it regulates the output voltage but does not shape the input current. Stage 4 (S-R flip-flop) includes the multiplier.

## 6. Two ways to generate the gate signal

### Triangular carrier (PWM)
A triangular wave at the switching frequency is compared with a control voltage `V_control`.
- ON when V_control > triangle, otherwise OFF.
- Duty cycle D = V_control / V_triangle_peak.
- With a **constant** `V_control` the duty cycle is fixed (open loop). If `V_control` comes from a current PI, the duty cycle is computed automatically (stage 3).

### Peak current mode with an S-R flip-flop
- The **clock** sets the flip-flop: MOSFET ON, inductor current rises.
- When the inductor current reaches the **current reference**, the **comparator** resets the flip-flop: MOSFET OFF, current falls.
- The duty cycle is a result of where the current reaches the reference, so it is automatic (stage 4).

Why a flip-flop? It is a latch. After a reset, switching noise on the comparator cannot turn the MOSFET on again within the same cycle, and exactly one turn-on happens per clock period.

| | Triangle + current PI | Peak current mode (S-R) |
|---|---|---|
| Current PI needed | Yes | No |
| Noise immunity | Lower | Higher (latch) |
| Overcurrent protection | Extra logic needed | Inherent through the reference |
| Slope compensation | Not needed | Needed at duty above 50 % |

## 7. Sizing check

- Output power: 48 V x 16 A = 768 W.
- Approximate peak input current at 30 V rms: I_peak about 2 P / V_peak = 2 x 768 / 42.4 = 36 A, more once losses are included.
- Maximum inductor current slope is about V_in / L. With L = 1 mH and V_in = 42 V this is about 42 A/ms, enough to follow a sinusoidal reference of this size for most of the cycle.
