# Troubleshooting log

Errors and observations from building the models, with the fix for each.

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Simulation stops with "1000 consecutive zero crossing events" (`powergui/EquivalentModel/State-Space` and `Relational Operator`) | The comparator input (i_L minus i_ref) keeps crossing zero near the line-voltage zero crossings while a variable-step continuous solver is used | Switch powergui to **Discrete** (`1e-6`) and use a fixed-step solver |
| 2 | "The FixedStepDiscrete solver cannot be used ... because it contains continuous states" | The PI controller was continuous-time | Use a discrete PI (`PI(z)`, sample time `1e-6`) |
| 3 | "The computed sample time (5E-07) is not an integer multiple of the fixed step size (1E-06)" on the Pulse Generator | A 5 % pulse width of a 5e-5 s period is 2.5 us | Use 2 us (4 %) or sample-based mode: period 50, width 2, sample time `1e-6` |
| 4 | From tag has no source | The output-voltage Goto tag was renamed (`[A]` to `[D]`) but one From tag kept the old name | Rename both ends |
| 5 | Output voltage stuck low (about 32 V, 38 V, 43.5 V) while the voltage PI output sat at its limit (40 A, 60 A, 85 A) | Raising the limit only helped a little each time | Check that `[E]` peaks at 1.0, that source amplitude and gain agree (peak vs rms), and that the limit is realistic (about 45 to 50 A peak for 768 W) |
| 6 | PI output swings between about 37 A and 45 A at 100 Hz and touches the limit | The proportional term passes the 100 Hz output ripple into the current reference | Lower Kp, keep Ki, or low-pass filter the voltage feedback (10 to 20 Hz) |

## Checks that were useful
- A scope with a Mux of i_L (`[F]`), i_ref (Product output) and the gate (`[C]`), zoomed into one 10 ms rectified cycle: shows at once whether the current reaches the reference and whether the gate toggles.
- A scope on `[E]` to confirm the normalised voltage peaks at 1.
- Temporarily replacing the voltage PI by a constant current (for example 10 A) to test the current loop alone.
