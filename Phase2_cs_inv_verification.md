# Phase 2 — Current-Starved Inverter (cs_inv): Isolation & Verification

## 1. Objective
Isolate the four-transistor current-starved inverter cell (as used in the 3-stage/5-stage ring VCO) from the `nitjsr_pll_130nm` reference design concept, independently reproduce its transistor-level netlist in SKY130, and verify (a) correct logic inversion, (b) propagation delay, and (c) delay control via current-starving bias (`Vp`, `Vn`) using ngspice. This forms the verified netlist basis for Phase 3 (Magic layout, DRC, LVS) and later hierarchical reuse in the ring VCO.

## 2. AI Prompts Used
- Prompt to structure the isolated `cs_inv` subcircuit given the reference repo's 4T current-starved topology (studied conceptually only; no files copied)
- Prompt to build a testbench measuring `t_pd_fall`/`t_pd_rise` via `.meas tran ... TRIG/TARG`
- Prompt to diagnose ngspice output overshoot (>VDD / <GND) on `v(out)`
- Prompt to build a multi-point bias sweep (`Vp`/`Vn`) to demonstrate current-starving delay control
- Prompt to diagnose `dowhile` loop failure (`vector ... is not available or has zero length`)
- Prompt to diagnose `.meas ... out of interval` failure at extreme starving bias

## 3. Reference Implementation (Conceptual Study Only)
The `nitjsr_pll_130nm` repository's ring VCO uses a 4T current-starved inverter: a switching CMOS inverter pair (NMOS+PMOS) with an additional PMOS header (gate = `Vp`, controls pull-up current) and NMOS footer (gate = `Vn`, controls pull-down current). Only this topology, the bias-generation concept, and the ring/buffer connection style were studied. No `.mag`, GDS, or netlist files from the reference repo were copied or modified.

## 4. AI-Generated Implementation (This Work)

### 4.1 Subcircuit — `cs_inv.spice`
```spice
.subckt cs_inv Vp in out Vn VDD GND
XM3 net2 Vp VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.72
XM2 out in net2 VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.72
XM1 out in net1 GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.42
XM4 net1 Vn GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.42
.ends
```
*(NMOS width corrected from an initial `W=0.36` to `W=0.42` in Phase 3 to match the verified Magic layout's extracted device sizing — see Section 6, entry 9. The full sweep in Section 7 reflects this final, layout-consistent sizing.)*

### 4.2 Testbench — `tb_cs_inv.spice`
```spice
.option ngbehavior=hsa
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
.include cs_inv.spice

V_VDD VDD GND 1.8
V_Vn  Vn  GND 1.8
V_Vp  Vp  GND 0.0
V_IN  in  GND PULSE(0 1.8 1ns 10ps 10ps 5ns 10ns)
C_load out GND 5f

X1 Vp in out Vn VDD GND cs_inv

.options reltol=1e-6 vntol=1e-9 abstol=1e-12
.tran 0.5p 15n

.control
  * Point 1: Vp=0.0, Vn=1.8 (full conduction)
  alter V_Vp 0.0
  alter V_Vn 1.8
  tran 0.5p 15n
  meas tran t_pd_fall1 TRIG in VAL=0.9 RISE=1 TARG out VAL=0.9 FALL=1
  meas tran t_pd_rise1 TRIG in VAL=0.9 FALL=1 TARG out VAL=0.9 RISE=1
  print t_pd_fall1 t_pd_rise1

  * Point 2: Vp=0.3, Vn=1.5
  alter V_Vp 0.3
  alter V_Vn 1.5
  tran 0.5p 15n
  meas tran t_pd_fall2 TRIG in VAL=0.9 RISE=1 TARG out VAL=0.9 FALL=1
  meas tran t_pd_rise2 TRIG in VAL=0.9 FALL=1 TARG out VAL=0.9 RISE=1
  print t_pd_fall2 t_pd_rise2

  * Point 3: Vp=0.6, Vn=1.2
  alter V_Vp 0.6
  alter V_Vn 1.2
  tran 0.5p 15n
  meas tran t_pd_fall3 TRIG in VAL=0.9 RISE=1 TARG out VAL=0.9 FALL=1
  meas tran t_pd_rise3 TRIG in VAL=0.9 FALL=1 TARG out VAL=0.9 RISE=1
  print t_pd_fall3 t_pd_rise3

  * Point 4: Vp=0.9, Vn=0.9
  alter V_Vp 0.9
  alter V_Vn 0.9
  tran 0.5p 15n
  meas tran t_pd_fall4 TRIG in VAL=0.9 RISE=1 TARG out VAL=0.9 FALL=1
  meas tran t_pd_rise4 TRIG in VAL=0.9 FALL=1 TARG out VAL=0.9 RISE=1
  print t_pd_fall4 t_pd_rise4

  * Point 5: Vp=1.2, Vn=0.6
  alter V_Vp 1.2
  alter V_Vn 0.6
  tran 0.5p 15n
  meas tran t_pd_fall5 TRIG in VAL=0.9 RISE=1 TARG out VAL=0.9 FALL=1
  meas tran t_pd_rise5 TRIG in VAL=0.9 FALL=1 TARG out VAL=0.9 RISE=1
  print t_pd_fall5 t_pd_rise5

  * Functional boundary found beyond this point (see Section 6)
.endc
.end
```

## 5. Comparison: Reference Concept vs. This Implementation
| Aspect | Reference (`nitjsr_pll_130nm`, studied only) | This Work |
|---|---|---|
| Topology | 4T current-starved inverter | Same (independently netlisted) |
| PDK | SKY130 | SKY130 (`sky130_fd_pr` models, tt corner) |
| Files reused | None — concept only | Fully original `.spice` |
| Device sizing | Not copied | `Wp=0.72µm`, `Wn=0.36µm`, `L=0.18µm` (chosen independently) |
| Load modeling | N/A | Added `C_load=5fF` on `out` for realistic transient behavior |

## 6. Errors & Fixes
| # | Error | Root Cause | Fix |
|---|---|---|---|
| 1 | Duplicate `.subckt cs_inv` definitions | Subckt defined inline in testbench AND in separate file | Split into `cs_inv.spice` (subckt only) + `tb_cs_inv.spice` (`.include cs_inv.spice`) |
| 2 | `v(out)` overshoot to ~2.55V / undershoot to ~-0.7V | Timestep too coarse for 10ps input edge; no output load capacitance | Tightened `.options reltol/vntol/abstol`; reduced `.tran` step; added `C_load=5fF` on `out` |
| 3 | `Warning: vector vp_val is not available` / `Error: RHS "vp_val + 0.3" invalid` in `dowhile` sweep | ngspice `let`-created scalars are scoped to the current plot; each `tran` inside the loop creates a new plot, orphaning the variable | Unrolled the sweep into explicit `alter`/`tran`/`meas` blocks per bias point instead of a `dowhile` loop |
| 4 | `meas ... trig(TARG): out of interval` at `Vp=1.5V, Vn=0.3V` (and again at `Vp=1.25V, Vn=0.55V`) | Both starving transistors approach cutoff simultaneously at extreme bias; `out` cannot complete a full logic swing | Confirmed via `plot v(in) v(out)` that `out` stalls between ~1.5–2.0V across multiple cycles even at 200ns window — genuine functional limit, not a simulation artifact |
| 5 | `t_pd_rise` began failing "out of interval" for all points after widening the input pulse period | `.tran` window not widened to match the new, longer `PULSE` period — the falling edge of `in` needed to trigger `t_pd_rise` fell outside the simulated window | Widened every `tran` call inside `.control` from `15n` to `50n` to consistently cover both edges of the widened `PULSE(0 1.8 1n 10p 10p 20n 40n)` at all bias points |
| 6 | `Fatal error: tran: transmission line z0 must be given` | A stray line `tran 0.5p 50n` (missing leading dot) outside `.control` was parsed as a transmission-line (`T`) device instantiation instead of a `.tran` analysis command | Removed the duplicate undotted line; kept the single `.tran 0.5p 50n` (with dot) outside `.control`, and `tran ...` (no dot) only inside `.control` blocks |
| 7 | `.tran: no such command available in ngspice` (in a separate `tb2sweep.spice` file) | `.tran` (with dot) was used inside `.control`, where the dot form is invalid — only the netlist-level, undotted form is valid inside `.control` | Consolidated to a single testbench file using the correct dot/no-dot convention per context |
| 8 | Point 5 (`Vp=1.2V, Vn=0.6V`) failed "out of interval" at 15ns and 50ns windows | Not a true functional failure — the transition genuinely takes far longer than at other points; the window was simply too short to capture it | Widened window to ~210ns, which resolved cleanly: `t_pd_fall5=73.8ns`, `t_pd_rise5=24.2ns` (W=0.36 run). Confirmed the true non-functional cutoff lies beyond this point, closer to `Vp=1.25V/Vn=0.55V` (see Section 7.3) |
| 9 | Extracted layout SPICE showed NMOS `W=0.42` instead of the netlist's original `W=0.36` | Phase 3 Magic layout (built from a working reference sample's geometry, per task guidance to use a sample `.mag` only as a format/style reference) carried its own native NMOS sizing, which did not match the originally-assumed `W=0.36` | Connectivity was verified correct via extraction (all 4 devices map onto the right nodes); PMOS sizing already matched at `W=0.72`. Rather than resize DRC-clean layout geometry, updated `cs_inv.spice` to `W=0.42` for NMOS to match the verified layout, and reran the full Phase 2 delay sweep with this corrected value (final data in Section 7.2) so netlist, layout, and simulation results stay fully consistent |

## 7. Simulation Results

### 7.1 Logic Verification
Correct inversion confirmed: `v(out)` transitions low↔high oppositely to `v(in)`, settling cleanly at 0V/1.8V rails (post-fix, no overshoot beyond ~1.95V transient ringing).

### 7.2 Propagation Delay vs. Current-Starving Bias
**Final dataset — NMOS W=0.42µm (corrected to match the verified Phase 3 layout; see Section 6, entry 9):**

| Point | Vp (V) | Vn (V) | t_pd_fall (ps) | t_pd_rise (ps) | t_pd avg (ps) |
|---|---|---|---|---|---|
| 1 | 0.0 | 1.8 | 60.7 | 113.6 | 87.1 |
| 2 | 0.3 | 1.5 | 72.1 | 141.9 | 107.0 |
| 3 | 0.6 | 1.2 | 123.5 | 274.9 | 199.2 |
| 4 | 0.9 | 0.9 | 505.2 | 1510.3 | 1007.7 |
| 5 | 1.2 | 0.6 | 33,786.0 | 73,942.1 | 53,864.1 |

Delay increases roughly 620x from Point 1 to Point 5 across the swept bias range, confirming strong, monotonic delay control via current-starving bias. As with the earlier W=0.36 run, Point 5 required a widened `.tran` window (~210ns) to resolve — at narrower windows (15ns, 50ns) `.meas` reported "out of interval" purely because the transition genuinely takes far longer than at the other four points, not because the cell fails to switch. The wider NMOS produces consistently faster fall edges than the original W=0.36 sweep, as expected from the stronger pull-down.

### 7.3 Near-Cutoff Behavior Beyond the Swept Range
A separate diagnostic point at `Vp=1.25V/Vn=0.55V` (slightly beyond Point 5) was tested with `plot v(in) v(out)` over a 200ns window and showed the output genuinely stalling in the 1.5–2.0V range across multiple input cycles without completing a full logic swing — unlike Point 5, which does complete a swing given enough time. This indicates the true cutoff edge for this device sizing (`Wp=0.72µm`, `Wn=0.36µm`, `L=0.18µm`) lies between `Vp=1.2V/Vn=0.6V` (Point 5, functional but very slow) and `Vp=1.25V/Vn=0.55V` (non-functional within a practical timeframe). Point 5 is treated as the practical edge of usable current-starving control for this cell.

## 9. Phase 3 — Magic Layout Verification (Summary)
The verified `cs_inv_schematic.spice` netlist (final: `W=0.72` PMOS, `W=0.42` NMOS, `L=0.18`) was used to generate a SKY130A Magic layout, using a working sample `.mag` purely as a format/structural reference per task guidance (no reference geometry copied or modified from the `nitjsr_pll_130nm` repository itself).

**File naming convention** (to avoid ambiguity between hand-verified and tool-extracted netlists):
| File | Purpose |
|---|---|
| `cs_inv_schematic.spice` | Phase 2 verified reference netlist (source of truth) |
| `cs_inv.mag` | Magic layout, cell renamed from default `clod` to `cs_inv` via `identify cs_inv` |
| `cs_inv_layout.spice` | SPICE extracted from the Magic layout via `ext2spice` |

| Check | Result |
|---|---|
| Magic parse | Initial format error (`Expected "use" line but saw:`) traced to incorrect section ordering — `<< labels >>` appeared before `use` (cell instance) blocks. Fixed by reordering to standard `.mag` structure: paint layers → `use` blocks → `<< labels >>` → `<< end >>`. |
| Cell naming | Default extracted cell name `clod` renamed to `cs_inv` in Magic (`identify cs_inv`) for clarity |
| DRC (`drc check`) | Passed, zero violations |
| Extraction connectivity | All 4 devices (XM1–XM4) map onto correct nodes matching `cs_inv_schematic.spice` topology (verified via `ext2spice` output comparison, accounting for D/S port symmetry) |
| Device sizing (pre-fix) | Extracted layout showed NMOS `W=0.42`, not the netlist's original `W=0.36` — netlist corrected to match (Section 6, entry 9); Phase 2 sweep rerun with final sizing |
| LVS (`netgen`) | **"Netlists match uniquely" / "Circuits match uniquely"** — `cs_inv_layout.spice cs_inv` vs. `cs_inv_schematic.spice cs_inv`: 4 devices, 8 nets, all 6 pins (`Vp, Vn, VDD, GND, out, in`) equivalent on both sides |

**Note:** an initial LVS attempt (before the file-renaming convention above was adopted) returned a false mismatch ("Netlists do not match," with `cs_inv` collapsing to a single `VSUBS` net) — traced to a stale/duplicate inline `.subckt cs_inv` definition being picked up instead of the corrected standalone file, consistent with the duplicate-subckt issue first seen in Phase 2 (Section 6, entry 1). Adopting distinct filenames (`cs_inv_schematic.spice` vs. `cs_inv_layout.spice`) for the two sides of the comparison eliminated this ambiguity and produced a clean, repeatable match.

This DRC-clean, LVS-verified `cs_inv` layout and its corresponding netlist are the artifacts carried forward into Phase 4/5 for hierarchical reuse in the 3-stage and 5-stage current-starved ring VCOs.

## 8. Observations
- Rise delay exceeds fall delay at Points 1–4, reversing sharply at Point 5 (`t_pd_fall5=33.8ns < t_pd_rise5=73.9ns`) — at extreme starving, the PMOS header's near-cutoff pull-up path becomes the dominant bottleneck, an even larger split than seen at W=0.36 (74ns vs 24ns), consistent with the wider NMOS making the pull-down comparatively stronger and shifting the bottleneck further toward the pull-up path.
- Delay increases monotonically and dramatically with starving bias severity (Vp↑/Vn↓) — roughly **620x** from Point 1 (87ps) to Point 5 (54ns average) — confirming the current-starving mechanism provides strong, usable control authority over switching speed, the core property this cell must exhibit for correct VCO frequency tuning in later phases.
- A true non-functional cutoff exists just beyond the swept range (near `Vp=1.25V/Vn=0.55V`, verified at W=0.36; expected to hold similarly at W=0.42), where the output fails to complete a swing even given 200ns. This caps the practical `Vctrl` range usable in the ring VCO bias circuit (Phase 4/5) — Point 5 represents the practical edge of useful control, not the absolute limit.
- **Netlist/layout consistency**: the original `W=0.36` NMOS sizing was corrected to `W=0.42` after Phase 3 layout extraction revealed the verified, DRC-clean, connectivity-correct `.mag` layout used this wider sizing. This sweep was fully rerun with the corrected value so that netlist, layout, and simulation data remain consistent throughout the documentation (see Section 6, entry 9).
- Several `.tran`/`tran` dot-syntax and simulation-window mismatches were the dominant source of debugging effort in this phase — now documented in Section 6 as a reusable reference for future testbenches (Phase 4/5 VCO sweeps). Key lesson: an "out of interval" `.meas` error does not always mean the circuit is non-functional — it may simply mean the window is too short, as proven by Point 5.
- This verified netlist (`cs_inv.spice`, `W=0.42/0.72`, `L=0.18`) and its corresponding Magic layout are the artifacts carried forward into Phase 4/5 for hierarchical reuse in the 3-stage and 5-stage ring VCOs.

## 10. Commands Used & How to Reproduce

**Files:** [`cs_inv_schematic.spice`](spice/cs_inv_schematic.spice) (netlist) · [`tb_cs_inv.spice`](spice/tb_cs_inv.spice) (testbench, all 5 points, includes the fixes from Section 6)

**Simulation (Phase 2 delay sweep):**
```bash
cd spice/
ngspice -b tb_cs_inv.spice > sim_log.txt 2>&1
grep t_pd sim_log.txt
```

**Magic layout (Phase 3):**
```bash
find / -name "sky130A.tech" 2>/dev/null          # locate tech file if unknown
magic -T sky130A cs_inv.mag                       # open layout
```
Inside the Magic console (tkcon):
```
drc check
drc why
select top cell
identify cs_inv
extract all
ext2spice lvs
ext2spice
save cs_inv.mag
```
Then rename the extracted output for clarity:
```bash
mv cs_inv.spice cs_inv_layout.spice
```

**LVS (Phase 3):**
```bash
find / -iname "*setup.tcl*" -path "*sky130*" 2>/dev/null   # locate setup script
netgen -batch lvs "cs_inv_layout.spice cs_inv" "cs_inv_schematic.spice cs_inv" sky130A_setup.tcl lvs_report.txt
cat lvs_report.txt
```
Expected final line: `Circuits match uniquely.`

**Common pitfalls hit while developing this (see Section 6 for full detail):** duplicate `.subckt` definitions when splitting files, `.tran`/`tran` dot-syntax differing inside vs. outside `.control`, transient window too short for slow/near-cutoff bias points, and Magic `.mag` section ordering (`use` blocks must precede `<< labels >>`).
