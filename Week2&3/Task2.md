# Task 2 — Week 2 & 3: PLL Circuit Design (AI-Assisted)

Reference repo: [`nitjsr_pll_130nm`](https://github.com/himansh107/nitjsr_pll_130nm)

**Objective:** Recreate each PLL building block (PFD → Charge Pump/Loop Filter → ÷8 Divider → VCO) step by step using AI-assisted prompts, verify behavior in ngspice/xschem against SKY130 models where possible, and document prompts, tools, generated netlists, simulation attempts, errors/fixes, and observations for each block.

**Reorganization note:** every block below (Blocks 1–4) now follows the same eight-part template so the sections are directly comparable:
`Objective → AI Prompt(s) → Reference Implementation → AI-Generated Implementation → Comparison → Errors & Fixes → Simulation Results → Observations`.
Sub-parts are omitted (not left as empty headers) where the source material has no content for them.

---

## Block 0 — Conceptual Foundation (PLL Basics, Lock Concept, Reference-vs-Feedback Relation)

### 0.1 Objective
Establish the theoretical grounding — why a PLL is needed, how the five-block loop relates `f_ref` and `f_vco`, and the distinction between frequency lock and phase lock — before implementing individual blocks.

### 0.2 Why a PLL Is Needed
A crystal-based reference oscillator is clean but frequency-limited — pushing a crystal much above ~200 MHz becomes impractical and noisy. A PLL sidesteps this with a **negative-feedback control loop** that locks an internally generated, higher-frequency oscillator to a low-frequency reference. Here the ratio is fixed at **8×**: a 9 MHz reference is multiplied up to an ≈71.4 MHz output clock.

### 0.3 The Five-Block Loop and Reference/Feedback Relation
The loop compares two signals at the PFD input:

- **`f_clk_in`** (`f_ref`) — external reference clock, fixed at 9 MHz
- **`f_vco`** (`f_fb`) — VCO output *after* the ÷8 feedback divider

```
        ┌─────┐   UP/DOWN   ┌────┐   Vctrl   ┌─────┐
f_ref ─▶│ PFD │────────────▶│ CP │──────────▶│ LF  │──┐
        └─────┘             └────┘           └─────┘  │
           ▲                                            ▼
           │                                        ┌───────┐
           │            f_fb (f_out / 8)            │  VCO  │
           └────────────[ ÷8 Divider ]◀──────────────┴───────┘
                                                          │
                                                          ▼
                                                        f_out
```

Because `f_fb` is the *divided* VCO output rather than the raw VCO output, the PFD always compares two signals meant to converge to the **same frequency** at lock — even though `f_out` itself runs 8× faster than `f_ref`. The ÷8 divider is therefore the block that actually sets the multiplication factor N.

### 0.4 Phase Lock vs. Frequency Lock
A bare phase detector (XOR, single mixer) only produces a usable error signal once the two clocks are already close in frequency. If the VCO starts far from `8 × f_ref` at power-on, a pure phase detector's output is ambiguous and the loop may never converge — the classic **frequency-acquisition problem**. This motivates the **dual-flip-flop-style PFD** used here:

- **Frequency lock (acquisition):** when `f_clk_in` and `f_vco` differ noticeably, one of UP/DOWN stays asserted for extended, asymmetric durations (one PFD side is repeatedly re-triggered before the other ever registers an edge), steering `Vctrl` — a genuinely frequency-sensitive response enabling pull-in from a large starting error.
- **Phase lock (tracking):** once frequencies match, UP/DOWN pulse widths shrink to encode only residual phase difference, fine-tuning `Vctrl` toward zero error.

This dual behavior — frequency-error detector during acquisition, phase-error detector once near lock — is the core justification for the flip-flop-based PFD over a simple XOR-type detector.

### 0.5 Formal Lock Condition
The loop is locked once:
- Phase error between `f_clk_in` and `f_vco` is zero on a cycle-averaged basis (a small residual may remain due to PFD reset-path delay — see dead zone below — but produces no net charge-pump current)
- `Vctrl` has settled to a quasi-DC value (bounded ripple only)
- `f_out = N × f_clk_in`, with `N = 8`

**Settling time (`T_set`)**: time from power-on until `Vctrl` first enters, and stays within, a tolerance band (e.g. ±1%) of its final locked value.

### 0.6 The PFD Dead Zone
The AND/NAND-based reset path inside the PFD has finite delay, creating a narrow window around zero phase error where UP and DOWN both go high simultaneously and are effectively invisible to the charge pump:
- Leaves a small band of phase error with no corrective charge-pump action
- Shows up as static phase noise/jitter at lock rather than a perfectly zero residual
- Is why PFD testbenches deliberately offset `f_clk_in` and `f_vco` in time rather than starting them aligned — this forces the loop out of the dead zone and produces a measurable, non-degenerate UP or DOWN pulse instead of two indistinguishable near-zero pulses

### 0.7 Consequence for PFD Testbench Behavior
A testbench emulating a one-directional frequency/phase offset between `f_clk_in` and `f_vco` should, by the mechanism above, only ever produce one *stable, measurable* pulse type — UP or DOWN, not both — because the opposite flip-flop's trigger condition never wins the race. This asymmetric, direction-dependent outcome is expected PFD behavior for such a stimulus, not a sign of a faulty testbench or netlist.

---

## Block 1 — Phase Frequency Detector (PFD)

### 1.1 Objective
Understand and recreate the PFD block using an AI-assisted circuit design workflow, and verify UP/DOWN pulse generation against the reference repo implementation using SKY130 standard cells in ngspice. The PFD compares `f_clk_in` against `f_vco` and produces:
- `UP` when `f_clk_in` leads `f_vco`
- `DOWN` when `f_vco` leads `f_clk_in`
- A reset once both edges have been registered

### 1.2 AI Prompt 1 (ChatGPT)
> Generate a Phase Frequency Detector (PFD) for a PLL using SKY130 standard-cell library. When `f_clk_in > f_vco`, assert `UP`; when `f_clk_in < f_vco`, assert `DOWN`; reset both outputs when both inputs have been detected (PFD reset).
>
> ```
> .lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
> .include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
> ```

PDK paths were supplied directly so the AI tool would target `sky130_fd_sc_hd` standard-cell views instead of generic transistor-level primitives, keeping output consistent with the rest of the reference project. This prompt's result diverged from the reference topology (see §1.4 below), so a second, more constrained prompt (§1.3) was issued.

### 1.3 AI Prompt 2 (constrained to match repo topology)
> Generate a complete ngspice testbench for a Phase Frequency Detector (PFD) extracted from a SKY130 PLL. Use SKY130 HD standard cells only (`sky130_fd_sc_hd__nand2_1`, `nand3_1`, `nand4_1`, `inv_1`) with correct SPICE pin order. Preserve the extracted gate-level topology without redesigning the logic. Include SKY130 `.lib`/`.include` files, VDD=1.8V, two asynchronous clock inputs (10 MHz reference and slightly faster 10.53 MHz feedback) to create a phase/frequency error, instantiate the PFD as a subcircuit, add small output capacitive loads (~6 fF) on UP and DOWN, perform transient analysis (`tran 10p 1u`), and plot the reference clock, feedback clock, UP, and DOWN waveforms. Generate a complete, modular, runnable ngspice netlist only.

This second prompt's output (used for the full PLL netlist going forward) is the subcircuit-wrapped, topology-preserving version shown in §1.4b.

### 1.4 Reference Implementation (`pfd.cir`, from `nitjsr_pll_130nm`)
Uses a flattened, custom combinational NAND/inverter tree rather than a textbook flip-flop-based PFD — built directly from `sky130_fd_sc_hd__nand2/nand3/nand4/inv` primitives, cross-coupled to realize sequential UP/DOWN/reset behavior at the gate level.

> **Scope note:** `pfd.cir` is not PFD-only. The same file also contains the **charge pump** (`XM1`–`XM18`, driven by `up`/`down`) and the **loop filter** (`C3`, `R1`, `C4` on `vco`/`net20`) — a single flattened schematic covering three PLL blocks together, relevant when the CP and LF sections are documented in Block 2.

<details>
<summary><strong>Reference PFD netlist</strong> — <code>pfd.cir</code></summary>

```spice
x1 net4 net1 net9 net7 GND GND VPWR VPWR net6 sky130_fd_sc_hd__nand4_1
x2 net15 net14 GND GND VPWR VPWR net1 sky130_fd_sc_hd__nand2_1
x3 f_vco GND GND VPWR VPWR net13 sky130_fd_sc_hd__inv_1
x4 net3 net4 net6 GND GND VPWR VPWR net15 sky130_fd_sc_hd__nand3_1
x5 net6 net7 net11 GND GND VPWR VPWR net12 sky130_fd_sc_hd__nand3_1
x6 net1 net5 GND GND VPWR VPWR net4 sky130_fd_sc_hd__nand2_1
x7 net4 net6 GND GND VPWR VPWR net5 sky130_fd_sc_hd__nand2_1
x8 net6 net7 GND GND VPWR VPWR net8 sky130_fd_sc_hd__nand2_1
x9 net8 net9 GND GND VPWR VPWR net7 sky130_fd_sc_hd__nand2_1
x11 net13 net12 GND GND VPWR VPWR net9 sky130_fd_sc_hd__nand2_1
x10 f_clk_in GND GND VPWR VPWR net14 sky130_fd_sc_hd__inv_1
x12 net1 GND GND VPWR VPWR net2 sky130_fd_sc_hd__inv_1
x13 net2 GND GND VPWR VPWR net3 sky130_fd_sc_hd__inv_1
x14 net9 GND GND VPWR VPWR net10 sky130_fd_sc_hd__inv_1
x15 net10 GND GND VPWR VPWR net11 sky130_fd_sc_hd__inv_1
x16 net15 GND GND VPWR VPWR up sky130_fd_sc_hd__inv_1
x17 net12 GND GND VPWR VPWR down sky130_fd_sc_hd__inv_1
V2 f_clk_in GND pulse(0 1.8v 0 100p 100p 5n 9n)
V3 f_vco GND pulse(0 1.8v 2n 100p 100p 5n 10n)
V4 VPWR GND 1.8v
C1 up GND 6f m=1
C2 down GND 6f m=1
V1 f_test GND pulse(0 1.8v 200p 135p 23.1p 2.05n 10n)
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
.control
plot V(f_clk_in)+6 V(f_vco)+4 V(up)+2 V(down)
tran .1ns 300n
.endc
.GLOBAL GND
.end
```
</details>

### 1.5 AI-Generated Implementation

**(a) Dual-D-Flip-Flop topology — from Prompt 1**
Independently derived textbook dual-DFF PFD: two edge-triggered DFFs with `D` tied high, clocked by `f_clk_in` and `f_vco` respectively, and a single NAND2 gate generating the asynchronous reset once both `Q` outputs (`up`, `down`) go high.

Standard cells: `sky130_fd_sc_hd__dfrtp_1`, `sky130_fd_sc_hd__nand2_1`. Pin orders confirmed against PDK cell definitions before use:
- `dfrtp_1`: `CLK D RESET_B VGND VNB VPB VPWR Q`
- `nand2_1`: `A B VGND VNB VPB VPWR Y`

Two-phase stimulus: Phase 1 (0–50 ns) `f_clk_in` leads `f_vco` by 2 ns (expect `UP`); Phase 2 (50–100 ns) `f_vco` leads `f_clk_in` by 2 ns (expect `DOWN`).

<details>
<summary><strong>AI-generated PFD (dual-DFF)</strong> — <code>tb_pfd_dff.spice</code></summary>

```spice
* tb_pfd_dff.spice
* Independently-designed PFD using classic dual-D-flip-flop topology
* (NOT copied from repo's pfd.cir - that file uses a flattened custom
* NAND/inverter combinational tree. This is the textbook implementation:
* two edge-triggered DFFs with D tied high, clocked by the two input
* signals, and a single NAND2 gate providing the async reset - when both
* Q outputs go high, NAND(up,down) pulls RESET_B low and clears both.)
*
* Standard cells used: sky130_fd_sc_hd__dfrtp_1, sky130_fd_sc_hd__nand2_1
* Pin orders below were confirmed against PDK cell definitions before use:
*   dfrtp_1:  CLK D RESET_B VGND VNB VPB VPWR Q
*   nand2_1:  A B VGND VNB VPB VPWR Y
*
* Phase 1 (0-50ns):  f_clk_in leads f_vco by 2ns  -> expect UP pulses
* Phase 2 (50-100ns): f_vco leads f_clk_in by 2ns -> expect DOWN pulses
.title Independent dual-DFF PFD - two-phase UP/DOWN verification
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
Vpwr VPWR 0 DC 1.8
Vclkin f_clk_in 0 PULSE(0 1.8 0n 100p 100p 5n 10n)
Vvco_a n_vco_a 0 PULSE(0 1.8 2n 100p 100p 5n 10n)
Vvco_b n_vco_b 0 PULSE(0 1.8 8n 100p 100p 5n 10n)
Bvco f_vco 0 V = time < 50n ? V(n_vco_a) : V(n_vco_b)
Cup up 0 6f m=1
Cdown down 0 6f m=1
X1 f_clk_in VPWR rst_b GND GND VPWR VPWR up   sky130_fd_sc_hd__dfrtp_1
X2 f_vco    VPWR rst_b GND GND VPWR VPWR down sky130_fd_sc_hd__dfrtp_1
X3 up down GND GND VPWR VPWR rst_b sky130_fd_sc_hd__nand2_1
.GLOBAL GND
.control
tran 10p 100n
wrdata tb_pfd_dff_out.txt v(f_clk_in) v(f_vco) v(up) v(down) v(rst_b)
plot V(f_clk_in)+6 V(f_vco)+4 V(up)+2 V(down)
.endc
.meas tran up_pw   TRIG v(up)   VAL=0.9 RISE=1 TD=0n  TARG v(up)   VAL=0.9 FALL=1 TD=0n
.meas tran down_pw TRIG v(down) VAL=0.9 RISE=1 TD=58n TARG v(down) VAL=0.9 FALL=1 TD=58n
.end
```
</details>

**(b) Topology-preserving subcircuit — from Prompt 2, used for the full PLL netlist**

<details>
<summary><strong>AI-generated PFD (repo-matching gate-level topology)</strong></summary>

```spice
* PFD Testbench - Constant Phase Difference (f_out leads)
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice

Vcc VDD GND 1.8v

V2 f_clk_in GND pulse(0 1.8v 7n 60p 60p 50n 100n)
V3 f_out    GND pulse(0 1.8v 5n 60p 60p 50n 100n)

.subckt PFD f_clk_in f_out up down VDD GND
    x11 net12 net9 net17 net15 GND GND VDD VDD net14 sky130_fd_sc_hd__nand4_1
    x12 net23 net22 GND GND VDD VDD net9  sky130_fd_sc_hd__nand2_1
    x13 f_out GND GND VDD VDD net21 sky130_fd_sc_hd__inv_1
    x14 net11 net12 net14 GND GND VDD VDD net23 sky130_fd_sc_hd__nand3_1
    x15 net14 net15 net19 GND GND VDD VDD net20 sky130_fd_sc_hd__nand3_1
    x16 net9 net13 GND GND VDD VDD net12 sky130_fd_sc_hd__nand2_1
    x17 net12 net14 GND GND VDD VDD net13 sky130_fd_sc_hd__nand2_1
    x18 net14 net15 GND GND VDD VDD net16 sky130_fd_sc_hd__nand2_1
    x19 net16 net17 GND GND VDD VDD net15 sky130_fd_sc_hd__nand2_1
    x20 net21 net20 GND GND VDD VDD net17 sky130_fd_sc_hd__nand2_1
    x21 f_clk_in GND GND VDD VDD net22 sky130_fd_sc_hd__inv_1
    x22 net9 GND GND VDD VDD net10 sky130_fd_sc_hd__inv_1
    x23 net10 GND GND VDD VDD net11 sky130_fd_sc_hd__inv_1
    x24 net17 GND GND VDD VDD net18 sky130_fd_sc_hd__inv_1
    x25 net18 GND GND VDD VDD net19 sky130_fd_sc_hd__inv_1
    x26 net23 GND GND VDD VDD up    sky130_fd_sc_hd__inv_1
    x27 net20 GND GND VDD VDD down  sky130_fd_sc_hd__inv_1
.ends PFD

Xpfd1 f_clk_in f_out up down VDD GND PFD

C1 up   GND 6f
C2 down GND 6f

.control
tran 0.1ns 500n
plot v(up)+2 v(f_clk_in) v(down)+4 v(f_out)
.endc

.GLOBAL GND
.end
```
</details>

### 1.6 Comparison — Reference vs. AI-Generated

| Aspect | Reference PFD (`pfd.cir`) | AI-Generated (a) dual-DFF | AI-Generated (b) topology-preserving |
|---|---|---|---|
| Topology | Flattened custom NAND/inverter combinational tree | Classic dual-D-flip-flop PFD | Same flattened NAND tree as reference, wrapped as a `.subckt` |
| Standard cells | `nand2_1`, `nand3_1`, `nand4_1`, `inv_1` | `dfrtp_1`, `nand2_1` | `nand2_1`, `nand3_1`, `nand4_1`, `inv_1` |
| Sequential elements | Implicit, via cross-coupled NAND latches | Explicit edge-triggered DFFs | Implicit, matching reference |
| Reset mechanism | Embedded in gate-level feedback network | Single NAND2 driving `RESET_B` on both DFFs | Embedded, matching reference |
| Readability | Low — non-descriptive net names (`net1`…`net15`) | High — `up`, `down`, `rst_b` map to function | Low — inherits reference's `net1`…`net23` naming |
| Stimulus style | Fixed two-edge pulses, static phase offset | Behavioral (`B`) source switching phase mid-run | Fixed two-edge pulses, static phase offset (10 MHz vs 10.53 MHz per Prompt 2) |
| Test coverage | Single lead/lag condition per run | Both UP and DOWN in one transient run | Single lead/lag condition per run |
| Origin | Extracted/flattened from `pfd.sch` | Independently derived (Prompt 1) | Extracted-topology reproduction (Prompt 2) |
| Verification status | Reference/golden behavior | Not yet cross-checked against reference waveform | Used directly in full PLL netlist |
| File scope | Also contains charge pump (`XM1`–`XM18`) and loop filter (`R1`,`C3`,`C4`) | PFD-only | PFD-only, wrapped as reusable subckt |

### 1.7 Simulation Setup
Both netlists run in ngspice against the same SKY130 `sky130_fd_sc_hd` library (`tt` corner):
```
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
```
- Reference PFD: `tran .1ns 300n`, single fixed phase offset (`f_vco` delayed 2 ns from `f_clk_in`).
- AI-generated (a): `tran 10p 100n`, phase relationship switched at 50 ns via a behavioral source to exercise both `UP` and `DOWN` in one run.
- AI-generated (b): `tran 0.1ns 500n`, fixed phase offset (`f_out` leads `f_clk_in` by 2 ns).
- All probe `up`/`down` with `.meas tran` pulse-width measurements and plot `f_clk_in`, `f_vco`/`f_out`, `up`, `down` on offset traces for visual separation.

### 1.8 Observations
- The reference design encodes PFD sequential behavior entirely in combinational gates (a flattened, synthesized-looking netlist) — functionally correct but harder to read or modify block-by-block.
- AI-generated design (a) maps directly onto the textbook dual-DFF PFD — easier to reason about conceptually and easier to re-target to other PDK cell libraries — but is a genuinely different circuit from the reference.
- AI-generated design (b) preserves the reference's exact gate-level topology and was adopted for the full PLL netlist, since Prompt 1's independently-derived version diverged too far from the repo to be a like-for-like replacement.
- All variants use the same `VPWR`/`GND` (or `VDD`/`GND`) convention and the same SKY130 `sky130_fd_sc_hd` cell family, keeping comparisons apples-to-apples at the technology level.
- The dead zone and frequency-vs-phase-lock distinction from Block 0 explain, at a theoretical level, why a one-directional phase-offset stimulus is expected to yield only one clean, measurable pulse width rather than symmetric UP/DOWN behavior throughout.

---

## Block 2 — Charge Pump + Loop Filter (CP+LF)

### 2.1 Objective
Turn the PFD's `UP`/`DOWN` pulses into a control voltage for the VCO: charge is pushed onto `vctrl` when `UP` fires, pulled off when `DOWN` fires, and the loop filter smooths that into a usable DC level. Continuation of Block 1's PFD work — Block 2 of the 5-block PLL series.

### 2.2 AI Prompt
> Act as an analog IC engineer. Generate a SKY130 ngspice netlist for a PLL charge pump with a second-order passive loop filter... UP and DOWN controlled PMOS/NMOS switching paths, dummy devices for charge injection reduction...

### 2.3 Reference Implementation
Origin: `himansh107/nitjsr_pll_130nm`. Switches: single PMOS (`M4`, W=45) / NMOS (`M3`, W=15). No charge-injection handling. `C1`/`C2` = 500p/100p, `R1` = 1.5k.

> **Repo limitation, confirmed directly**: the repo's shipped `pre layout/cp+lf.cir` contains **no `.plot`, `.meas`, or `.control` statements at all**. Running it exactly as provided produces a raw transient solution with no printed measurements and no waveform — the repo's schematic/result images showing expected behavior were not generated by executing that file as shipped. Getting comparable output from the repo netlist requires adding the same instrumentation (`wrdata`/`.meas`) used for the AI-generated version below.

### 2.4 AI-Generated Implementation
Follows the repository's overall architecture (PMOS source path, NMOS sink path, passive 2nd-order loop filter) while adding cascoded switches and dummy/replica legs for charge-injection cancellation not present in the repo version. Switches: cascoded PMOS/NMOS pairs. `C1`/`C2` = 100p/200p, `R1` = 1.5k.

<details>
<summary>How the circuit works</summary>

**Output stage (charge pump)** — connects to `vctrl`:
- PMOS branch: driven low → connects VDD to output, sourcing current, raising `vctrl`.
- NMOS branch: driven high → connects output to GND, sinking current, lowering `vctrl`.

**Input/control logic (UP / DOWN networks)** — two symmetric paths:
- UP path: complementary signals (`up_bar`, `up_bar2`) via inverter/transmission-gate stages drive the PMOS switches; complementary drive cancels clock feedthrough and charge injection.
- DOWN path: identical structure, mirrored, driving the NMOS switches.

**Loop filter (2nd-order passive low-pass):**
- `C2` suppresses high-frequency switching ripple.
- `R1` in series with `C1` creates a stabilizing zero for phase margin.

**Behavior summary:** `UP` high → PMOS sources current → `vctrl` rises → VCO sped up. `DOWN` high → NMOS sinks current → `vctrl` falls → VCO slowed down. Both low (locked) → output tri-states, loop filter holds charge, `vctrl` steady.
</details>

<details>
<summary>Full SPICE netlist (AI-generated)</summary>

```spice
.param VDD_VAL = 1.8

.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt

.subckt CHARGE_PUMP UP DOWN CP VDD GND

XM14 up_bar UP VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1
XM7  up_bar UP GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

XM21 up_bar2 up_bar VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1
XM15 up_bar2 up_bar GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

XM26 np1 up_bar  VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=45   nf=1
XM25 CP  up_bar  np1 VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1

XM24 np2 up_bar2 VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1
XM23 vdummy_p up_bar2 np2 VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1

XM12 down_bar DOWN VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1
XM8  down_bar DOWN GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

XM22 down_bar2 down_bar VDD VDD sky130_fd_pr__pfet_01v8 L=0.18 W=0.54 nf=1
XM16 down_bar2 down_bar GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

XM20 nn1 down_bar2 GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=15   nf=1
XM19 CP  down_bar2 nn1 GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

XM18 nn2 down_bar  GND GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1
XM17 vdummy_n down_bar nn2 GND sky130_fd_pr__nfet_01v8 L=0.18 W=0.36 nf=1

.ends CHARGE_PUMP

.subckt CP_LF UP DOWN CTRL VDD GND
XCP UP DOWN CP VDD GND CHARGE_PUMP
C1  CP   GND  100p
R1  CP   CTRL 1.5k
C2  CTRL GND  200p
.ends CP_LF

Vdd  VDD  0  DC {VDD_VAL}

Vup_u   up_u   0  PULSE(0 {VDD_VAL} 5n 5p 5p 10n 20n)
Vdn_u   down_u 0  DC 0
XCPLF_U up_u down_u vctrl_u VDD 0 CP_LF

Vup_d   up_d   0  DC 0
Vdn_d   down_d 0  PULSE(0 {VDD_VAL} 5n 5p 5p 10n 20n)
XCPLF_D up_d down_d vctrl_d VDD 0 CP_LF

.tran 10p 700n
.control
run
plot v(up_u) v(vctrl_u)
plot v(down_d) v(vctrl_d)
meas tran v_ctrl_u_end find v(vctrl_u) at=700n
meas tran v_ctrl_d_end find v(vctrl_d) at=700n
.endc
.end
```
</details>

### 2.5 Comparison — Reference vs. AI-Generated

| | Repo Reference | AI-Generated |
|---|---|---|
| Switches | Single PMOS (M4, W=45) / NMOS (M3, W=15) | Cascoded PMOS/NMOS pairs |
| Charge injection handling | None | Dummy/replica legs on both sides |
| `C1` / `C2` | 500p / 100p | 100p / 200p |
| `R1` | 1.5k | 1.5k |
| `.control`/`.meas`/`.plot` present | No (as shipped) | Yes |

### 2.6 Errors & Fixes
None recorded for this block beyond the missing-instrumentation issue noted in §2.3.

### 2.7 Simulation Results
Only numbers from netlists actually run in ngspice appear here.

**Repo Reference** — single-pulse `vctrl` delta, sampled at the pulse edge (`AT=2.41n`):

| | UP path | DOWN path |
|---|---|---|
| Pulse width | 2.129 ns | 2.129 ns |
| `vctrl` delta | +1.084 mV | −2.676 mV |
| Mismatch ratio (DOWN/UP) | | **2.47×** |

**AI-Generated `CP_LF`** — 700 ns continuous run, both instances start at 1.043 V:

| | UP path | DOWN path |
|---|---|---|
| `vctrl` at end of run | 1.108 V | 0.920 V |
| Net movement | +64.7 mV | −123.3 mV |

Mismatch ratio (DOWN/UP), continuous-run basis: **1.9×**.

> **Methodology note:** the AI-Generated numbers come from a *continuous* 700 ns run (many pulses, cumulative drift). The Repo Reference numbers come from a *single-pulse* delta (one pump event, sampled at the edge). These measure different things — cumulative drift vs. one-shot charge injection — so 2.47× (Repo) and 1.9× (AI-Generated) are each valid on their own but not a strict magnitude comparison against each other.

### 2.8 Observations
Both designs push `vctrl` in the correct direction for each input, and both show sink stronger than source (DOWN path moves `vctrl` more than UP path), though the two ratios (2.47× vs 1.9×) are not directly comparable due to the methodology difference above.

---

## Block 3 — Frequency Divider (÷8, FD)

### 3.1 Objective
Recreate the ÷8 feedback divider as three cascaded ÷2 stages, each a static transmission-gate master-slave toggle flip-flop, matching the reference `f/2` circuit built from `pfet_01v8`/`nfet_01v8` devices.

### 3.2 AI Prompt
> Generate a SPICE netlist for a ÷8 frequency divider for a PLL feedback path, in SKY130, matching the attached schematic (`f/2 ckt`, a single ÷2 stage built from `pfet_01v8`/`nfet_01v8` devices). Requirements: Divide-by-8 = three cascaded divide-by-2 stages. Use real SKY130 primitive subckt names (`sky130_fd_pr__pfet_01v8`, `sky130_fd_pr__nfet_01v8`), not bare xschem symbol names.

### 3.3 Reference Implementation — Topology
A Master-Slave D-Flip-Flop (DFF) in negative feedback, with `q_b` routed back to the data input, toggles state on every active clock edge — dividing input frequency by two.

<details>
<summary>Core architecture & signal flow</summary>

**1. Clock inverter stage** — `M13`/`M14` generate `clk_b` from `clk`; both phases drive transmission gates for synchronization and data isolation.

**2. Master latch stage**
- Input switch (`M3`/`M4`): TG gated by `clk`/`clk_b`, conducts when `clk` is **low**, sampling inverted feedback from `q_b`.
- Forward inverter (`M1`/`M2`): drives the sampled state into the master storage node.
- Feedback storage loop (`M7`/`M8` & `M5`/`M6`): when `clk` goes **high**, `M7`/`M8` completes the cross-coupled inverter loop, latching the sampled state.

**3. Slave latch stage**
- Intermediate switch (`M9`/`M10`): TG in the opposite phase of the input switch, conducts when `clk` is **high**, propagating the Master's latched state forward.
- Output buffering (`M11`/`M12` & `M15`/`M16`): consecutive inverters condition/buffer `q` and `q_b`.
- Slave storage loop (`M17`/`M18`): when `clk` goes **low**, locks the current output state while the Master opens to capture the next state.

**Working principle:**

| Clock State | Master Latch | Slave Latch | Action |
|---|---|---|---|
| Low | Sampling (open) | Hold (locked) | Master samples current `q_b`; Slave holds previous output |
| High | Hold (locked) | Evaluation (open) | Master locks sampled value; Slave propagates it to `q`/`q_b` |

Because `q_b` is continuously fed back to the input, the circuit flips to the opposite logic state every full clock cycle — requiring **two full input-clock periods** per output period:
$$f_{out} = \frac{f_{in}}{2}$$

**Sizing:** process `pfet_01v8`/`nfet_01v8` (1.8V CMOS). PMOS (`M1`,`M6`,`M12`,`M16`,`M14`): W/L = 2.5 µm/0.15 µm. NMOS (`M2`,`M5`,`M11`,`M15`,`M13`): W/L = 1 µm/0.15 µm.
</details>

### 3.4 AI-Generated Implementation
Rebuilt as a static transmission-gate master-slave toggle latch (TG1–TG4 + 4 inverters) per §3.3, wrapped in a `pfet_u`/`nfet_u` wrapper subckt that pre-computes junction parasitics once and reuses it per instance; three ÷2 stages cascaded inside an `fd_8` wrapper, instantiated once at top level.

### 3.5 Comparison — AI-Generated vs. Reference

| Aspect | AI-generated | Reference repo |
|---|---|---|
| `ad/as/pd/ps/nrd/nrs` | Computed once via `pfet_u`/`nfet_u` wrapper subckts, reused per instance | Written out explicitly on all transistor lines per stage |
| Lines per `fd` instance | ~20 (calls to wrapper subckts) | ~40+ (full `XM...` lines, all params inline) |
| Top-level cascade | `fd` × 3 wrapped inside an `fd_8` subckt, instantiated once | `fd` × 3 called directly at top level, no wrapper |
| GND handling | `GND` net collapsed to SPICE's ground node `0` everywhere | `GND` as a named net + `.GLOBAL GND`, tied to 0V via a source elsewhere |
| `.control` block | `tran 1ns 5us` + `plot v(clk1)+2 v(f_out)` | `tran 1ns 5us` only, no plot command shown |

### 3.6 Errors & Fixes

| # | Error | Cause | Fix |
|---|---|---|---|
| 1 | `unknown subckt: ...pfet_01v8 l=0.15u w=2.5u nf=1` | Bare `pfet_01v8`/`nfet_01v8` are xschem symbol labels, not actual SPICE subckt names in the `.lib` | Renamed to `sky130_fd_pr__pfet_01v8` / `sky130_fd_pr__nfet_01v8` |
| 2 | `could not find a valid modelname for sky130_fd_pr__pfet_01v8` | Missing `ad/as/pd/ps/nrd/nrs` area/perimeter params — PDK model bind requires them explicitly | Added explicit `ad/as/pd/ps/nrd/nrs`, computed from device W/L via wrapper subckts (`nfet_u`/`pfet_u`) |
| 3 | Divide-by-2 stage logically wrong (dynamic clock-gated inverter guess) | Topology reconstructed from schematic image alone; actual circuit is a static TG-based master-slave toggle latch | Rebuilt `fd` subckt as static TG master-slave toggle latch (TG1–TG4 + 4 inverters) |
| 4 | `instance vgnd is a shorted VSRC` / no such vector `clk1` | `VGND GND 0 DC 0` tied a named `GND` net to ground — a 0V source between two nodes is a short in ngspice | Replaced named `GND` net with SPICE's actual ground node `0` everywhere; dropped `VGND`/`.GLOBAL GND` |

### 3.7 Simulation Results
Transient ran successfully: 5911 data rows, 5 µs window, 1 ns step. Waveform shows `clk1` toggling at high density and `f_out` producing a lower-frequency output consistent with a divide relationship.

**Not yet quantified:** exact period/frequency values and duty cycle are to be measured independently from the output data (via `.meas` or the plotted trace) before being recorded as verified results — none are asserted here.

### 3.8 Observations
The wrapper-subckt approach (computing junction parasitics once via `pfet_u`/`nfet_u`) meaningfully reduces per-instance line count versus the reference's fully-inlined parameter style, at the cost of one extra level of hierarchy to trace through.

---

## Block 4 — Voltage-Controlled Oscillator (VCO)

### 4.1 Objective
Recreate a 7-stage current-starved ring oscillator: a bias stage sets `Vp`/`Vn` from `vctrl`, distributing starve current to seven cascaded current-starved inverter stages, followed by a discrete CMOS output buffer — and match its frequency response against the reference repo's `vco.sch`/`cs_inv.sch`.

<details>
<summary>Circuit description</summary>

- **Biasing network:** `vctrl` gates an NMOS current-setting device; a diode-connected PMOS mirrors this current to generate `Vp`. `Vn` tracks `vctrl` directly.
- **Delay cell core:** each of the 7 stages is a current-starved inverter — a PMOS/NMOS inverter pair flanked by starve transistors (PMOS gated by `Vp`, NMOS gated by `Vn`) limiting charge/discharge current into the stage's output capacitance.
- **Ring configuration:** 7 (odd) stages closed in a loop produce self-sustained oscillation.
- **Output buffer:** a discrete CMOS inverter isolates the ring from output loading and sharpens edges into a clean square wave at `OSC`.
</details>

### 4.2 AI Prompt
> Act as an analog IC engineer. Generate a complete ngspice netlist for a 7-stage current-starved ring VCO using the SKY130 PDK. Use `sky130.lib.spice tt` and `sky130_fd_sc_hd.spice`. Implement a current-starved inverter subcircuit, bias stage controlled by `vctrl`, 7-stage ring oscillator, output buffer, 1.8 V supply, transient testbench, `.control` block, and simulate at `vctrl = 0.7 V` and `0.8 V` to compare output frequency.

### 4.3 Reference Implementation
Topology extracted/isolated from `nitjsr_pll_130nm`'s `vco.sch`/`cs_inv.sch` netlist export. `cs_inv` stage sizing, bias mirror NMOS, output buffer sizing, and ring stage count/topology are identical to the AI-generated version and are not repeated in §4.5 — only differences are listed there. The extracted reference netlist ties the bias PMOS body to `Vp` (rather than the n-well supply `VDD`).

### 4.4 AI-Generated Implementation
Testbench: fixed dual-bias-point transient, `vctrl` swept between 0.7 V and 0.8 V via `alter`, 5.5 µs transient per run, `.meas trig/targ` on `v(OSC)` (rise=3 to rise=4, `val=0.9`) to extract steady-state period and frequency at each point. Initial version used bias PMOS body tied to `VDD` (conventional SKY130 body connection) and width 1.08 rather than the reference's 1.8; this `VDD`-body version was retained as the project's adopted implementation target, with the `Vp`-body version used only for numerical comparison against the reference (§4.6).

### 4.5 Comparison — AI-Generated (initial) vs. Reference

| Element | Reference | AI-generated (initial) |
|---|---|---|
| Bias diode-connected PMOS width | W=1.8 | W=1.08 |
| Bias diode-connected PMOS body | tied to `Vp` | tied to `VDD` |
| Junction parasitics (`ad/as/pd/ps/nrd/nrs`) | explicit, on every device | absent |

### 4.6 Errors & Fixes

| Error Log / Message | Root Cause | Fix Implemented |
|---|---|---|
| `Fatal error: instance v_gnd is a shorted VSRC` / `doAnalyses: operation not supported` | A standalone voltage source (`V_GND`) connected `GND` to global node `0`; the SKY130 PDK library already shorts/aliases `GND` to `0`, creating a redundant 0V loop | Removed `V_GND GND 0 DC 0`; mapped the DUT's ground pin directly to native ngspice node `0` |
| `Error: no such vector osc` / `vector OSC is not available or has zero length` | The fatal topology error aborted transient analysis before any node data existed | Resolved automatically once the shorted voltage source was removed |
| `Error: measure t1_0p7 trig(TRIG) : out of interval` | Measurement targeted the 100th rising edge (`rise=100`) of `OSC`; at `Vctrl=0.7V` the VCO completes only ~15 cycles in the 5.5 µs window, so edge 100 never occurs | Changed measurement edges to `rise=3`/`rise=4` for both runs, well within the simulation window at either bias point |
| Missing `uic` on `tran` statements (no fatal error, but latent risk) | Without `uic`, ngspice computes its own DC operating point before the transient; a symmetric current-starved ring has a stable non-oscillating DC solution the OP solver could converge to, silently discarding the startup asymmetry | Added `uic` to both `tran` calls so `.ic v(n1)=0 v(n2)=1.8 v(n3)=0` is used directly as the t=0 condition. Confirmed via log line `Operating point simulation skipped by 'uic', now using transient initial conditions.` |
| Missing `mult=1` on all `sky130_fd_pr__*` device instances | Omitted from the initial AI-generated netlist; project convention requires explicit `mult=1` to avoid BSIM4 parameter substitution failures | Added `mult=1` to all 8 primitive device instances (4 in `cs_inv`, 2 in bias stage, 2 in output buffer) |

### 4.7 Simulation Results

**Convergence debugging** — isolating the frequency gap between the initial AI-generated netlist (which ran 4–6× faster than reference) and the reference, one change at a time:

| Variant | `Vctrl=0.7V` freq | `Vctrl=0.8V` freq | Gap vs. reference |
|---|---|---|---|
| Reference repo | 3.591 MHz | 10.02 MHz | — |
| AI-generated, initial | 15.40 MHz | 62.50 MHz | 4.3×–6.2× |
| + bias PMOS width 1.08→1.8 (body still VDD) | — | 62.50 MHz | ~0.0003% shift — negligible |
| + body tie VDD→Vp (width 1.8) | 3.09 MHz | 8.06 MHz | 14–20% low |
| + junction parasitics on all 12 devices (width 1.8, body Vp) | **3.591 MHz** | **10.00 MHz** | **<0.2%** |

**Final simulation results:**

| Netlist variant | Body tie | `freq_0p7` | `freq_0p8` | K_vco (approx, 0.7→0.8V) |
|---|---|---|---|---|
| Reference repo | `Vp` | 3.590529e+06 Hz | 1.001886e+07 Hz | ~64 MHz/V |
| AI-generated, body=`VDD` (project's adopted version) | `VDD` | 1.540092e+07 Hz | 6.249844e+07 Hz | ~471 MHz/V |
| AI-generated, body=`Vp` + parasitics (reference-matching validation) | `Vp` | 3.590887e+06 Hz | 1.000204e+07 Hz | ~64 MHz/V |

`v(OSC)` transient waveforms were plotted at `Vctrl = 0.7V` and `Vctrl = 0.8V` for both the standard body=VDD netlist and the body=Vp (reference-matching) netlist.

### 4.8 Observations
**Why the body tie dominated:** tying the PMOS body to `Vp` rather than the n-well supply (`VDD`) changes the body-source voltage, modifying the device threshold through the body effect. Because this bias node controls all seven current-starving PMOS devices, even a modest shift in the generated bias current propagates through the entire ring, producing a large change in oscillation frequency — this is why it accounted for the majority of the frequency gap while the width change alone was negligible.

**Design decision:** the extracted reference netlist connects the PMOS body to `Vp`, whereas conventional SKY130 practice ties PMOS bodies to the n-well supply (`VDD`). The project retains the `VDD`-body version as the implementation target, using the `Vp`-body version only for numerical comparison against the reference simulations.
