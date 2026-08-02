# Phase 5 — 5-Stage Ring VCO: Pre-Layout Simulation

## 1. Objective
Verify sustained oscillation of the fresh 5-stage current-starved ring VCO (built from the verified `cs_inv` cell, the same `vco_bias` bias generator, and an output buffer) across the same swept `Vctrl` range used for the 3-stage VCO, and extract matching metrics for direct comparison.

## 2. Circuit Under Test
`vco5.spice` — 5 instances of `cs_inv` in a ring (`X1→X2→X3→X4→X5→X1`), driven by the same `vco_bias` generator topology as the 3-stage VCO, with an identical NMOS/PMOS output buffer tapping `net1` to drive `osc`.

## 3. Testbench
`tbvco5.spice` — identical structure and `.meas` pattern to `tbvco3.spice`, using the same 5 `Vctrl` points for direct comparability. Point 1's window was widened to 150ns (from 100ns) to accommodate the 5-stage ring's longer per-cycle delay at the lowest bias point.

## 4. Results

| Point | Vctrl (V) | Period (ns) | Freq (MHz) | Startup (ns) | Duty (%) | Avg Power (µW) |
|---|---|---|---|---|---|---|
| 1 | 0.75 | 13.801 | 72.5 | 12.86 | 51.8 | 8.50 |
| 2 | 0.90 | 3.231 | 309.5 | 4.41 | 54.7 | 35.41 |
| 3 | 1.20 | 1.133 | 882.6 | 1.47 | 54.2 | 127.12 |
| 4 | 1.50 | 0.781 | 1281.2 | 1.19 | 53.5 | 223.73 |
| 5 | 1.70 | 0.703 | 1423.0 | 1.12 | 53.3 | 271.58 |

All 5 points oscillated successfully on the first run — no window or bias-point adjustments were needed beyond the Point 1 window widening noted above.

## 5. Derived Metrics
- **Tuning range**: 72.5 MHz → 1423.0 MHz (~19.6x) across `Vctrl = 0.75V → 1.70V`
- **K_VCO** (local slope, Points 2→3, most linear region): ≈ (882.6 − 309.5) MHz / (1.2 − 0.9) V ≈ **1910 MHz/V** — notably lower than the 3-stage's ~3224 MHz/V, consistent with the 5-stage ring's longer loop delay compressing the achievable frequency range for the same bias swing.
- **Startup time**: drops sharply with increasing Vctrl (12.86ns → 1.12ns), same qualitative trend as the 3-stage VCO.
- **Duty cycle**: stable and close to 50%, 51.8%–54.7% — tighter spread than the 3-stage VCO's 53.3–58.2%.
- **Power**: nearly identical to the 3-stage VCO at matching Vctrl points (e.g. 271.6µW vs. 269.5µW at Vctrl=1.7V) — expected, since the same bias circuit drives both, and total switching energy per unit time is similar despite the different stage count and frequency.

## 6. Comparison with 3-Stage VCO (Pre-Layout)

| Metric | 3-Stage | 5-Stage |
|---|---|---|
| Freq range | 126.0 – 2404.9 MHz | 72.5 – 1423.0 MHz |
| Tuning ratio | ~19.1x | ~19.6x |
| K_VCO (Points 2→3) | ~3224 MHz/V | ~1910 MHz/V |
| Startup time range | 15.54 – 1.41 ns | 12.86 – 1.12 ns |
| Duty cycle range | 53.3 – 58.2% | 51.8 – 54.7% |
| Power @ Vctrl=1.7V | 269.50 µW | 271.58 µW |

Both VCOs show consistent, monotonic frequency increase with `Vctrl`, satisfying the task's core requirement. The 5-stage ring oscillates at roughly 55-60% of the 3-stage's frequency at matching bias points, consistent with the added loop delay from two extra stages, while tuning ratio and power draw remain comparable — a good internal consistency check between the two designs.

## 7. Observations
- No errors or window-size failures were encountered in this sweep beyond the Point 1 window widening — likely because the fixes discovered during the 3-stage sweep (duty cycle `FALL=3` correction, avoiding near-cutoff bias points) were applied proactively here.
- The 5-stage ring's tighter duty-cycle spread (51.8–54.7% vs. 53.3–58.2%) suggests slightly better rise/fall symmetry accumulates over more stages — worth noting as an observation, though not a large effect.
- This dataset forms the pre-layout baseline for the 5-stage VCO; the same testbench structure and Vctrl points will be reused post-layout (with parasitics) for direct comparison per the task's required comparison table.

## 8. Commands Used & How to Reproduce

**Files:** [`vco5.spice`](spice/vco5.spice) (circuit, includes `bias.spice` and `cs_inv_schematic.spice`) · [`tbvco5.spice`](spice/tbvco5.spice) (full 5-point sweep testbench)

```bash
cd spice/
ngspice -b tbvco5.spice > vco5_sweep_log.txt 2>&1
grep -E "startup|t_period|freq|duty|power" vco5_sweep_log.txt
```

**To view the oscillation waveform directly** (interactive, single bias point):
```bash
ngspice tbvco5.spice
```
then in the ngspice prompt: `plot v(osc) v(vctrl)`

**Note:** this sweep ran clean on the first attempt — all fixes discovered in the 3-stage sweep (Point 1 bias choice, duty-cycle `FALL=3`, batch-mode logging) were applied proactively here. If reproducing from scratch, see `Phase5_vco3_prelayout.md` Section 6 for the underlying pitfalls.
