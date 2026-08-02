# Phase 5 — 3-Stage Ring VCO: Pre-Layout Simulation

## 1. Objective
Verify sustained oscillation of the fresh 3-stage current-starved ring VCO (built from the verified `cs_inv` cell, `vco_bias` bias generator, and an output buffer) across a swept `Vctrl` range, and extract frequency, tuning range, startup time, duty cycle, and average power for later comparison against post-layout (extracted) simulation.

## 2. Circuit Under Test
`vco3.spice` — 3 instances of `cs_inv` in a ring (`X1→X2→X3→X1`), driven by a shared `vco_bias` generator, with an NMOS/PMOS output buffer tapping one ring node (`net1`) to drive `osc`.

## 3. Testbench
`tbvco3.spice` — sweeps `Vctrl` across 5 values, running a `.tran` per point and measuring startup time, period/frequency, duty cycle, and average supply current (converted to power).

Key `.meas` pattern per point:
```spice
meas tran startupN WHEN v(osc)=0.9 RISE=1
meas tran t_periodN TRIG v(osc) VAL=0.9 RISE=2 TARG v(osc) VAL=0.9 RISE=3
meas tran t_highN TRIG v(osc) VAL=0.9 RISE=2 TARG v(osc) VAL=0.9 FALL=3
meas tran i_avgN AVG i(V_VDD) FROM=<window>
let freqN = 1/t_periodN
let dutyN = 100*t_highN/t_periodN
let powerN = abs(i_avgN)*1.8
```

## 4. Results

| Point | Vctrl (V) | Vp (V) | Period (ns) | Freq (MHz) | Startup (ns) | Duty (%) | Avg Power (µW) |
|---|---|---|---|---|---|---|---|
| 1 | 0.75 | ~1.14* | 7.939 | 126.0 | 15.54 | 53.3 | 9.19 |
| 2 | 0.90 | 0.846 | 1.711 | 584.4 | 3.95 | 58.2 | 35.96 |
| 3 | 1.20 | 0.635 | 0.644 | 1551.6 | 1.74 | 56.9 | 126.17 |
| 4 | 1.50 | 0.478 | 0.458 | 2181.3 | 1.42 | 55.7 | 221.93 |
| 5 | 1.70 | 0.409 | 0.416 | 2404.9 | 1.41 | 55.5 | 269.50 |

*Vp at Point 1 interpolated from the bias-generator sweep; not directly logged in this run.

## 5. Derived Metrics
- **Tuning range**: 126.0 MHz → 2404.9 MHz (~19x) across `Vctrl = 0.75V → 1.70V`
- **K_VCO** (local slope, Points 2→3, most linear region): ≈ (1551.6 − 584.4) MHz / (1.2 − 0.9) V ≈ **3224 MHz/V**. Note the frequency-vs-Vctrl curve is nonlinear (steeper at low Vctrl, flattening at high Vctrl) — a single K_VCO figure is an approximation; the full curve should be plotted for the final report.
- **Startup time**: drops sharply with increasing Vctrl (15.54ns → 1.41ns), consistent with faster loop gain at weaker starving.
- **Duty cycle**: stable across the sweep, 53.3%–58.2% — no severe asymmetry.
- **Power**: increases monotonically with frequency (9.19µW → 269.5µW), consistent with dynamic power scaling with switching activity. Energy-per-cycle (power/freq) is roughly consistent (~73–115 fJ/cycle) across Points 2–5, a reasonable sanity check for a ring oscillator of this size.

## 6. Errors & Fixes (this phase)
| # | Error | Root Cause | Fix |
|---|---|---|---|
| 1 | Point at `Vctrl=0.6V` failed to oscillate (`osc` settled near 0V, all `.meas` "out of interval") | `Vp≈1.135V` at this bias sits too close to the Phase 2 functional boundary (`Vp≈1.2–1.25V`); insufficient loop gain to sustain oscillation | Replaced with `Vctrl=0.75V`, safely inside the proven functional range — oscillation confirmed |
| 2 | Duty cycle computed as negative (~ -42% to -45%) | Waveform starts high at `t=0`; `.meas ... TARG v(osc) VAL=0.9 FALL=2` targeted a fall edge that occurred *before* the `RISE=2` trigger chronologically, not after | Changed target to `FALL=3` so it captures the fall immediately following the `RISE=2` trigger; duty cycle values corrected to a sensible 53–58% range |
| 3 | Log file not created (`grep: vco3_sweep_log.txt: No such file or directory`) | Ran ngspice interactively without batch mode/output redirection | Used `ngspice -b tbvco3.spice > vco3_sweep_log.txt 2>&1` to capture full console output to a file for reliable `grep`-based extraction |

## 7. Observations
- The ring achieves sustained, monotonic frequency scaling across the full swept range, satisfying the task requirement that "frequency must increase consistently with control voltage."
- Waveform shape at `osc` is not a clean square wave — visibly rounded/S-curved rising edges, consistent with the current-starving mechanism verified in Phase 2 (`t_pd_rise > t_pd_fall` at all but the most extreme bias points). Edge sharpness is expected to improve (more square) at higher Vctrl and degrade further at lower Vctrl, approaching the Phase 2 functional boundary.
- This dataset forms the pre-layout baseline for the 3-stage VCO; the same testbench structure and Vctrl points will be reused post-layout (with parasitics) for direct comparison per the task's required comparison table (frequency, tuning range, K_VCO, startup time, duty cycle, average power, layout area).

## 8. Commands Used & How to Reproduce

**Files:** [`vco3.spice`](spice/vco3.spice) (circuit, includes `bias.spice` and `cs_inv_schematic.spice`) · [`tbvco3.spice`](spice/tbvco3.spice) (full 5-point sweep testbench)

```bash
cd spice/
ngspice -b tbvco3.spice > vco3_sweep_log.txt 2>&1
grep -E "startup|t_period|freq|duty|power" vco3_sweep_log.txt
```

**To view the oscillation waveform directly** (interactive, single bias point):
```bash
ngspice tbvco3.spice
```
then in the ngspice prompt: `plot v(osc) v(vctrl)`

**Pitfalls hit here (see Section 6 for full detail):** `Vctrl=0.6V` doesn't oscillate (too close to the Phase 2 functional boundary — use `0.75V` instead); duty-cycle `.meas` needs `FALL=3` not `FALL=2` when the waveform starts high at `t=0`; always use `ngspice -b ... > log.txt 2>&1` (not interactive mode) if you want a `grep`-able log file.
