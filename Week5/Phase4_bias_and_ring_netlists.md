# Phase 4 — Bias Circuit & Ring VCO Netlist Construction (3-Stage & 5-Stage)

## 1. Objective
Study the reference repository's bias circuit and ring-connection concept (`nitjsr_pll_130nm`-style VCO), then independently design and verify a fresh bias generator, and use it together with the verified `cs_inv` cell (Phase 2/3) to construct separate 3-stage and 5-stage current-starved ring VCO netlists at the schematic level, ready for pre-layout simulation (Phase 5).

## 2. AI Prompts Used
- Prompt to design a fresh bias circuit given the reference's self-biased current-mirror concept (studied from the reference's actual bias schematic netlist, not copied)
- Prompt to build a bias-circuit testbench sweeping `Vctrl` and verifying `Vp`/`Vn` respond correctly and oppositely
- Prompt to diagnose an `M`-vs-`X` device-prefix error (raw MOSFET line used instead of the required PDK subcircuit call)
- Prompt to construct 3-stage and 5-stage ring netlists hierarchically instantiating the verified `cs_inv` cell plus the bias generator and an output buffer

## 3. Reference Implementation (Conceptual Study Only)
The reference VCO's bias generator uses a self-biased current generator: an NMOS (gate tied to `Vctrl` via a series resistor) pulls current from a node also fed by a diode-connected PMOS from VDD; the node self-settles to a voltage (`Vp`) that balances the two currents. `Vn` is tied directly to `Vctrl`. The reference ring uses 7 stages of `cs_inv`, each sharing the same `Vp`/`Vn` bias nodes, closed in a loop, with a final NMOS/PMOS inverter pair used as an output buffer tapping one internal ring node. Only this topology and connection concept were studied from the reference's schematic-level netlist — no files were copied or reused.

## 4. AI-Generated Implementation (This Work)

### 4.1 Bias Circuit — `bias.spice`
```spice
* Fresh current-starved VCO bias generator (self-biased current mirror)
* Sized to match the verified cs_inv NMOS width (W=0.42, corrected in Phase 3)
.subckt vco_bias vctrl Vp Vn VDD GND
* Direct tie: control voltage sets Vn (footer bias) for all ring stages
R_tie Vn vctrl 1
* MB_N: pulls bias current from node Vp, controlled by Vn
XMB_N Vp Vn GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.42
* MB_P: diode-connected PMOS, sources current into Vp, self-biasing the node
XMB_P Vp Vp VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=1.8
.ends
```
Independent design choices: `XMB_N` width matched to the verified `cs_inv` NMOS sizing (`W=0.42`) rather than the reference's `W=0.36`, for internal consistency with Phase 2/3; `XMB_P` width (`W=1.8`) independently chosen for bias-node headroom; standard body-tie convention used (PMOS body→VDD, NMOS body→GND) rather than the reference's non-standard PMOS body→Vp tie, simplifying later layout/DRC.

### 4.2 Bias Circuit Verification — `tb_vco_bias.spice`
Swept `vctrl` from 0.4V to 1.6V via `.dc`, plotted `Vp` and `Vn` directly in ngspice.

**Results:**
| vctrl (V) | Vn (V) | Vp (V) |
|---|---|---|
| 0.4 | 0.400 | 1.447 |
| 0.7 | 0.700 | 1.026 |
| 1.0 | 1.000 | 0.769 |
| 1.3 | 1.300 | 0.577 |
| 1.6 | 1.600 | 0.439 |

`Vn` tracks `vctrl` directly (1Ω tie, as designed); `Vp` decreases monotonically as `vctrl` increases — confirming correct self-biasing direction (both starving transistors open together as `vctrl` rises, matching the coordinated Vp/Vn control validated in Phase 2's delay sweep).

**Cross-check against Phase 2 functional boundary:** at `vctrl=0.4V`, `Vp=1.447V` exceeds the Phase 2 functional boundary (`Vp≈1.2–1.25V`), indicating the practical usable `Vctrl` range for sustained ring oscillation is narrower than the full swept range — confirmed later in Phase 5 (Point 1 at `Vctrl=0.6V` failed to oscillate; `Vctrl=0.75V` was required).

### 4.3 3-Stage Ring VCO — `vco3.spice`
```spice
.subckt vco3 vctrl osc VDD GND
.include vco_bias.spice
.include cs_inv.spice

X_BIAS vctrl Vp Vn VDD GND vco_bias

X1 Vp net1 net2 Vn VDD GND cs_inv
X2 Vp net2 net3 Vn VDD GND cs_inv
X3 Vp net3 net1 Vn VDD GND cs_inv

XBUF_N osc net1 GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36
XBUF_P osc net1 VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.72
.ends
```

### 4.4 5-Stage Ring VCO — `vco5.spice`
```spice
.subckt vco5 vctrl osc VDD GND
.include vco_bias.spice
.include cs_inv.spice

X_BIAS vctrl Vp Vn VDD GND vco_bias

X1 Vp net1 net2 Vn VDD GND cs_inv
X2 Vp net2 net3 Vn VDD GND cs_inv
X3 Vp net3 net4 Vn VDD GND cs_inv
X4 Vp net4 net5 Vn VDD GND cs_inv
X5 Vp net5 net1 Vn VDD GND cs_inv

XBUF_N osc net1 GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36
XBUF_P osc net1 VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.72
.ends
```

Both rings reuse the identical `cs_inv` (Phase 2/3 verified) and `vco_bias` (Section 4.1) subcircuits hierarchically, differing only in stage count and ring node naming — satisfying the task's hierarchical-reuse requirement.

## 5. Comparison: Reference Concept vs. This Implementation
| Aspect | Reference (conceptual only) | This Work |
|---|---|---|
| Bias topology | Self-biased NMOS/diode-PMOS current mirror, `Vn=Vctrl` via resistor | Same concept, independently netlisted and sized |
| Bias device sizing | NMOS `W=0.36`, PMOS `W=1.8` | NMOS `W=0.42` (matched to Phase 3 layout), PMOS `W=1.8` |
| Body ties | PMOS body→Vp (non-standard) | PMOS body→VDD, NMOS body→GND (standard) |
| Ring stage count | 7 stages | 3-stage and 5-stage (per task requirement) |
| Files reused | None — concept only | Fully original `.spice` files |

## 6. Errors & Fixes
| # | Error | Root Cause | Fix |
|---|---|---|---|
| 1 | `could not find a valid modelname` for `sky130_fd_pr__nfet_01v8`/`pfet_01v8` in bias circuit | Used `M` device-prefix (raw MOSFET line, expects a `.model` card) instead of `X` (subcircuit call) — this PDK's devices are subcircuits, not raw models | Changed `MB_N`/`MB_P` to `XMB_N`/`XMB_P`, matching the working convention already used in `cs_inv.spice` |

## 7. Simulation Results Summary
See Section 4.2 for bias circuit `Vp`/`Vn` sweep results. Full ring-level oscillation results (frequency, startup, duty cycle, power) are documented separately in Phase 5 reports (`Phase5_vco3_prelayout.md`, `Phase5_vco5_prelayout.md`), which build directly on the netlists constructed here.

## 8. Observations
- The self-biased current-mirror topology, once independently reproduced and sized to match the verified `cs_inv` cell, produces the correct coordinated `Vp`/`Vn` control direction needed for monotonic frequency tuning — later confirmed at the full ring level in Phase 5.
- Standardizing body ties (rather than following the reference's non-standard PMOS body→Vp tie) is expected to simplify DRC/well-tap requirements in the upcoming bias circuit layout (Phase 4/5 layout step), at the cost of a minor deviation from the reference's exact biasing scheme — a reasonable independent design tradeoff.
- Both ring netlists are structurally identical aside from stage count, which will simplify layout work by allowing the same per-stage placement/routing pattern to be extended from 3 to 5 instances.
- These netlists (`bias.spice`, `vco3.spice`, `vco5.spice`) are the artifacts carried into Phase 5 pre-layout simulation and, subsequently, into Magic layout construction for both ring VCOs.

## 9. Commands Used & How to Reproduce

**Files:** [`bias.spice`](spice/bias.spice) · [`tb_vco_bias.spice`](spice/tb_vco_bias.spice) · [`vco3.spice`](spice/vco3.spice) · [`vco5.spice`](spice/vco5.spice)

**Bias circuit verification:**
```bash
cd spice/
ngspice tb_vco_bias.spice
```
This runs a `.dc` sweep and opens an ngspice plot window showing `Vp` (falling) and `Vn` (rising) vs. `vctrl`, and saves `vco_bias_sweep.ps` for the report. Convert to PNG if needed:
```bash
convert vco_bias_sweep.ps vco_bias_sweep.png
```

**Ring netlists** (`vco3.spice`, `vco5.spice`) are not run standalone — they're `.include`d by the Phase 5 testbenches. See `Phase5_vco3_prelayout.md` and `Phase5_vco5_prelayout.md` for the full sweep commands.

**Pitfall hit here:** using `M` device-prefix instead of `X` for SKY130 PDK subcircuit calls — see Section 6. This PDK's `sky130_fd_pr__nfet_01v8`/`pfet_01v8` are subcircuits, not raw `.model` cards, so every instantiation needs an `X` prefix.
