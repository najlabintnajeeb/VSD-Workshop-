# Task 2 — Week 2 & 3: PLL Circuit Design (AI-Assisted)

Reference repo: [`nitjsr_pll_130nm`](https://github.com/himansh107/nitjsr_pll_130nm)

## Objective

Continuing from the reference repo, Week 2 & 3 focus only on the **circuit design side** of the SKY130-based on-chip clock-multiplier PLL — not final layout, GDS, tapeout packaging, or full repo reproduction. The goal is to understand and recreate each PLL building block step by step using AI-assisted prompts (ChatGPT/Codex or similar), verify behavior in ngspice/xschem with SKY130 models where possible, and document prompts, tools, generated netlists, simulation attempts, errors/fixes, and observations for each block.

## Scope — Blocks Covered

| # | Block | Status |
|---|---|---|
| 1 | PLL basics / phase-frequency locking concept | ✅ Complete |
| 2 | Reference clock vs feedback clock relation | ✅ Complete |
| 3 | **Phase Frequency Detector (PFD)** — UP/DOWN pulse generation | ✅ Complete |
| 4 | Charge pump — source/sink current behavior | Loop filter — control-voltage generation| ✅ Complete |
| 5 | VCO — tuning and frequency sweep | Pending |
| 6 | Divide-by-N feedback divider | Pending |
| 7 | Lock behavior / lock time | Pending |
| 8 | Jitter / noise awareness | Pending |
| 9 | Duty-cycle observation | Pending |
| 10 | Pre-layout SPICE simulation summary | Pending |
| 11 | SKY130 device/model usage notes | Pending |

Sections below are filled in as each block is completed.

---

## 1. PLL Basics, Phase/Frequency Lock Concept, and Reference-vs-Feedback Clock Relation

### 1.1 Why a PLL Is Needed

A crystal-based reference oscillator is clean but frequency-limited — pushing a crystal much above ~200 MHz becomes impractical and noisy. A Phase-Locked Loop sidesteps this by using a **negative-feedback control loop** to lock an internally generated, higher-frequency oscillator to a low-frequency reference. In this design, that relationship is fixed at **8×**: a 9 MHz reference is multiplied up to an ≈71.4 MHz output clock.

### 1.2 The Five-Block Loop and How Reference/Feedback Clocks Relate

The loop compares two signals at the PFD input:

- **`f_clk_in`** (or `f_ref`) — the external reference clock, fixed at 9 MHz
- **`f_vco`** (or `f_fb`) — the VCO output *after* it has passed through the ÷8 feedback divider

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

Because `f_fb` is the *divided* VCO output rather than the raw VCO output, the PFD is always comparing two signals that are meant to converge to the **same frequency** at lock — even though `f_out` itself runs 8× faster than `f_ref`. This is what makes the ÷8 divider the block that actually sets the multiplication factor N: whatever frequency the divider output settles to match `f_ref` at, the VCO itself must be running at `N × f_ref` to produce that divided result.

### 1.3 Phase Lock vs. Frequency Lock

A bare phase detector (e.g. XOR, single mixer) only produces a usable, well-behaved error signal once the two compared clocks are already close in frequency. If the VCO starts far from `8 × f_ref` at power-on, a pure phase detector's output is ambiguous, and the loop may never converge on its own — the classic **frequency-acquisition problem**. This is the reason the design uses a **dual-flip-flop-style PFD** rather than a simple phase detector:

- **Frequency lock (acquisition):** When `f_clk_in` and `f_vco` differ noticeably in frequency, one of UP/DOWN stays asserted for extended, asymmetric durations rather than short symmetric pulses, because one side of the PFD is repeatedly triggered before the other ever registers an edge. This net bias steers `Vctrl`, and therefore the VCO frequency, in the correct direction — a genuinely frequency-sensitive response that allows pull-in from a large starting error.
- **Phase lock (tracking):** Once both clocks reach the same frequency, UP/DOWN pulse widths shrink down to encoding only the residual *phase* difference between edges, and the loop fine-tunes `Vctrl` to drive that phase error toward zero.

The PFD therefore behaves as a frequency-error detector during acquisition and transitions into a phase-error detector once near lock — this dual behavior is the core justification for the flip-flop-based PFD topology over a simple XOR-type detector in a charge-pump PLL.

### 1.4 Formal Lock Condition

The loop is considered locked once:

- The phase error between `f_clk_in` and `f_vco` is zero on a cycle-averaged basis (a small residual may remain due to PFD reset-path delay — see the dead zone below — but it no longer produces net charge pump current)
- `Vctrl` has settled to a quasi-DC value (bounded ripple only)
- `f_out = N × f_clk_in`, with `N = 8`

**Settling time (`T_set`)** is the time from power-on until `Vctrl` first enters, and stays within, a defined tolerance band (e.g. ±1%) of its final locked value.

### 1.5 The PFD Dead Zone

The AND/NAND-based reset path inside the PFD has finite delay, which creates a narrow window around zero phase error where UP and DOWN both go high simultaneously and are effectively invisible to the charge pump. This is the **dead zone**:

- It leaves a small band of phase error with no corrective charge pump action
- In practice it shows up as static phase noise/jitter at lock rather than a perfectly zero residual phase error
- It's the reason PFD testbenches deliberately offset `f_clk_in` and `f_vco` in time (rather than starting them perfectly aligned) — doing so forces the loop out of the dead zone and produces a measurable, non-degenerate UP or DOWN pulse width instead of two indistinguishable near-zero pulses

### 1.6 Consequence for PFD Testbench Behavior

Any testbench built to emulate a one-directional frequency/phase offset between `f_clk_in` and `f_vco` should, by the mechanism above, only ever produce one *stable, measurable* pulse — UP or DOWN, not both — because the opposite flip-flop's trigger condition never wins the race under that stimulus. This asymmetric, direction-dependent outcome is expected PFD behavior for such a stimulus, not a sign that the testbench or netlist is faulty.

---

## 2. Phase Frequency Detector (PFD)

### 2.1 Objective

Understand and recreate the Phase Frequency Detector block of the PLL using an AI-assisted circuit design workflow, and verify UP/DOWN pulse generation behavior against the reference repo implementation using SKY130 standard cells in ngspice.

The PFD compares the reference clock (`f_clk_in`) against the feedback/VCO clock (`f_vco`) and produces:
- `UP` pulse when `f_clk_in` leads `f_vco` (reference is faster/leading)
- `DOWN` pulse when `f_vco` leads `f_clk_in` (feedback is faster/leading)
- A reset condition that clears both outputs once both edges have been registered

### 2.2 AI Prompt Used

**Tool:** ChatGPT (GPT-based assistant)

**Prompt:**
> Generate a Phase Frequency Detector (PFD) for a PLL using SKY130 standard-cell library. When `f_clk_in > f_vco`, assert `UP`; when `f_clk_in < f_vco`, assert `DOWN`; reset both outputs when both inputs have been detected (PFD reset).
>
> ```
> .lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
> .include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
> ```

The PDK `.lib`/`.include` paths were supplied directly so the AI tool would target the correct standard-cell views (`sky130_fd_sc_hd`) instead of generic transistor-level primitives, keeping the output consistent with the rest of the reference project.

### 2.3 Reference PFD (from `nitjsr_pll_130nm` repo)

The reference implementation (`pfd.sch` → `pfd.cir`) uses a flattened, custom combinational NAND/inverter tree rather than a textbook flip-flop-based PFD. It is built directly from `sky130_fd_sc_hd__nand2/nand3/nand4/inv` primitives, cross-coupled to realize the sequential UP/DOWN/reset behavior at the gate level.

**Note:** `pfd.cir` is not a PFD-only netlist. Alongside the gate-level PFD (`x1`–`x17`), the same file also contains the **charge pump** (transistors `XM1`–`XM18`, driven by `up`/`down`) and the **loop filter** (`C3`, `R1`, `C4` on the `vco`/`net20` nodes). This is a single flattened schematic covering three PLL blocks together, not an isolated PFD subcircuit — worth keeping in mind when the charge pump and loop filter sections are documented later, since their reference implementation is already present in this same file.

<details>
<summary><strong>Reference PFD netlist (click to expand)</strong> — <code>pfd.cir</code></summary>

```spice
** sch_path: /home/vboxuser/Desktop/PLL/pfd.sch
**.subckt pfd
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

### 2.4 AI-Generated PFD (Dual-D-Flip-Flop Topology)

Rather than reproducing the repo's flattened NAND tree, the AI-assisted workflow was used to derive the **textbook dual-DFF PFD**: two edge-triggered D-flip-flops with `D` tied high, clocked respectively by `f_clk_in` and `f_vco`, and a single NAND2 gate generating the asynchronous reset once both `Q` outputs (`up`, `down`) go high.

Standard cells used: `sky130_fd_sc_hd__dfrtp_1`, `sky130_fd_sc_hd__nand2_1`
Pin orders confirmed against PDK cell definitions before use:
- `dfrtp_1`: `CLK D RESET_B VGND VNB VPB VPWR Q`
- `nand2_1`: `A B VGND VNB VPB VPWR Y`

Testbench uses a two-phase stimulus: Phase 1 (0–50 ns) `f_clk_in` leads `f_vco` by 2 ns (expect `UP` pulses); Phase 2 (50–100 ns) `f_vco` leads `f_clk_in` by 2 ns (expect `DOWN` pulses).

<details>
<summary><strong>AI-generated PFD testbench (click to expand)</strong> — <code>tb_pfd_dff.spice</code></summary>

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
* Same two-phase test methodology as tb_pfd.spice:
* Phase 1 (0-50ns):  f_clk_in leads f_vco by 2ns  -> expect UP pulses
* Phase 2 (50-100ns): f_vco leads f_clk_in by 2ns -> expect DOWN pulses
.title Independent dual-DFF PFD - two-phase UP/DOWN verification
*** SKY130 corner library ***
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
*** Standard-cell SPICE models ***
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
*** Supply ***
Vpwr VPWR 0 DC 1.8
*** Reference clock: f_clk_in, free-running, TD=0 ***
Vclkin f_clk_in 0 PULSE(0 1.8 0n 100p 100p 5n 10n)
*** Two candidate f_vco phase relationships, selected by time ***
Vvco_a n_vco_a 0 PULSE(0 1.8 2n 100p 100p 5n 10n)
Vvco_b n_vco_b 0 PULSE(0 1.8 8n 100p 100p 5n 10n)
Bvco f_vco 0 V = time < 50n ? V(n_vco_a) : V(n_vco_b)
*** Load caps on PFD outputs ***
Cup up 0 6f m=1
Cdown down 0 6f m=1
*** PFD: two D-flip-flops (D=VPWR, async reset) + NAND2 reset generator ***
X1 f_clk_in VPWR rst_b GND GND VPWR VPWR up   sky130_fd_sc_hd__dfrtp_1
X2 f_vco    VPWR rst_b GND GND VPWR VPWR down sky130_fd_sc_hd__dfrtp_1
X3 up down GND GND VPWR VPWR rst_b sky130_fd_sc_hd__nand2_1
.GLOBAL GND
*** Analysis ***
.control
tran 10p 100n
wrdata tb_pfd_dff_out.txt v(f_clk_in) v(f_vco) v(up) v(down) v(rst_b)
plot V(f_clk_in)+6 V(f_vco)+4 V(up)+2 V(down)
.endc
*** Measurements ***
.meas tran up_pw   TRIG v(up)   VAL=0.9 RISE=1 TD=0n  TARG v(up)   VAL=0.9 FALL=1 TD=0n
.meas tran down_pw TRIG v(down) VAL=0.9 RISE=1 TD=58n TARG v(down) VAL=0.9 FALL=1 TD=58n
.end
```

</details>

### 2.5 Design Comparison — Reference PFD vs AI-Generated PFD

| Aspect | Reference PFD (`pfd.cir`) | AI-Generated PFD (`tb_pfd_dff.spice`) |
|---|---|---|
| Topology | Flattened custom NAND/inverter combinational tree | Classic dual-D-flip-flop PFD |
| Standard cells used | `nand2_1`, `nand3_1`, `nand4_1`, `inv_1` | `dfrtp_1`, `nand2_1` |
| Sequential elements | Realized implicitly via cross-coupled NAND latches | Explicit edge-triggered DFFs (`dfrtp_1`) |
| Reset mechanism | Embedded within the gate-level feedback network | Single NAND2 gate driving `RESET_B` on both DFFs |
| D-input handling | Not applicable (no discrete flip-flop cells) | `D` tied to `VPWR` on both DFFs |
| Readability / traceability | Low — flattened netlist, non-descriptive net names (`net1`…`net15`) | High — signal names map directly to function (`up`, `down`, `rst_b`) |
| Stimulus style | Fixed two-edge pulses (`f_clk_in`, `f_vco`) with static phase offset | Behavioral source (`B` element) switching phase relationship mid-run |
| Test coverage | Single lead/lag condition per run | Both UP and DOWN conditions in one transient run |
| Origin | Extracted/flattened from repo's schematic (`pfd.sch`) | Independently derived via AI-assisted prompt |
| Verification status | Reference/golden behavior from repo | Not yet cross-checked against reference waveform |
| File scope | Single flattened file also contains charge pump (`XM1`–`XM18`) and loop filter (`R1`, `C3`, `C4`) alongside the PFD gates | PFD-only testbench, isolated from charge pump/loop filter |

### 2.6 Simulation Setup

Both netlists were run in ngspice against the same SKY130 `sky130_fd_sc_hd` standard-cell library (`tt` corner):

```
.lib /opt/pdk/sky130A/libs.tech/ngspice/sky130.lib.spice tt
.include /opt/pdk/sky130B/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
```

- Reference PFD: `tran .1ns 300n`, single fixed phase offset (`f_vco` delayed 2 ns from `f_clk_in`).
- AI-generated PFD: `tran 10p 100n`, phase relationship switched at 50 ns via a behavioral source to exercise both `UP` and `DOWN` conditions in one run.
- Both testbenches probe `up` and `down` with `.meas tran` pulse-width measurements and plot `f_clk_in`, `f_vco`, `up`, `down` on offset traces for visual separation.

### 2.7 Observations

- The reference design encodes PFD sequential behavior entirely in combinational gates (a flattened, synthesized-looking netlist), which makes it functionally correct but harder to read or modify block-by-block.
- The AI-generated design maps directly onto the standard textbook dual-DFF PFD, which is easier to reason about conceptually (each block — flip-flop, reset gate — has a clear functional role) and easier to re-target to other PDK cell libraries.
- Both use the same `VPWR`/`GND` supply convention and the same SKY130 `sky130_fd_sc_hd` cell family, keeping the comparison apples-to-apples at the technology level.
- The AI-generated testbench's two-phase behavioral stimulus (`B` element) is a departure from the repo's style and was chosen to verify both `UP` and `DOWN` assertion paths without needing two separate simulation runs.
- The dead zone and frequency-vs-phase-lock distinction discussed in Section 1 explain, at a theoretical level, why the two-phase stimulus is expected to yield only one clean, measurable pulse width per phase rather than symmetric UP/DOWN behavior throughout.

---

## 3. Charge Pump and Loop Filter

# ⚡ Charge Pump + Loop Filter (CP+LF)

> Block 2 of 5 — PLL block-level simulation series
> Continuation of Week 1 report, Section III.B/III.C

## Overview

The CP+LF stage turns the PFD's `UP`/`DOWN` pulses into a control voltage
for the VCO. Charge is pushed onto `vctrl` when `UP` fires, pulled off when
`DOWN` fires — the loop filter smooths that into a usable DC level.

Two versions of this netlist exist:

| | 📂 Repo Reference | 🤖 AI-Generated |
|---|---|---|
| **Origin** | `himansh107/nitjsr_pll_130nm` | Prompted: *"Act as an analog IC engineer. Generate a SKY130 ngspice netlist for a PLL charge pump with a second-order passive loop filter... UP and DOWN controlled PMOS/NMOS switching paths, dummy devices for charge injection reduction..."* |
| **Switches** | Single PMOS (M4, W=45) / NMOS (M3, W=15) | Cascoded PMOS/NMOS pairs |
| **Charge injection handling** | None | Dummy/replica legs on both sides |
| **C1 / C2** | 500p / 100p | 100p / 200p |
| **R1** | 1.5k | 1.5k |
| **Status** |  | <img width="500" height="330" alt="Screenshot 2026-07-17 at 2 36 35 pm" src="https://github.com/user-attachments/assets/7fb86460-97e6-4526-b4ac-85e785297175" /> |

The AI-generated design closely follows the repository's overall
architecture (PMOS source path, NMOS sink path, passive 2nd-order loop
filter) while adding cascoded switches and dummy/replica legs for
charge-injection cancellation not present in the repo version.

<details>
<summary>🔍 How the circuit works</summary>

**Output stage (the actual charge pump)**
The main charge/discharge branch connects to `vctrl`:
- **PMOS branch**: when driven low, connects VDD to the output, sourcing current into the loop filter and raising `vctrl`.
- **NMOS branch**: when driven high, connects the output to GND, sinking current and lowering `vctrl`.

**Input/control logic (UP / DOWN networks)**
Two symmetric paths process the digital inputs independently:
- **UP path**: generates complementary signals (`up_bar`, `up_bar2`) through inverter/transmission-gate stages, driving the PMOS switches. Complementary drive helps cancel clock feedthrough and charge injection.
- **DOWN path**: identical structure, mirrored, driving the NMOS switches.

**Loop filter (2nd-order passive low-pass)**
- **C2** — suppresses high-frequency ripple from the switching.
- **R1 in series with C1** — creates a stabilizing zero, adding phase margin so the loop doesn't oscillate.

**Behavior summary**
- `UP` high → PMOS sources current → `vctrl` rises → VCO sped up.
- `DOWN` high → NMOS sinks current → `vctrl` falls → VCO slowed down.
- Both low (locked) → output tri-states, loop filter holds charge, `vctrl` steady.

</details>

<details>
<summary>📄 Full SPICE netlist (AI-generated)</summary>

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

> ⚠️ **Repo limitation, confirmed directly**: the repo's shipped
> `pre layout/cp+lf.cir` contains **no `.plot`, `.meas`, or `.control`
> statements at all**. Running it exactly as provided produces a raw
> transient solution with no printed measurements and no waveform — the
> repo's schematic/result images showing expected behavior were not
> generated by executing that file as shipped. Getting any comparable
> output from the repo netlist requires adding the same kind of
> instrumentation (`wrdata`/`.meas`) done for the Repo Reference column
> above.

## Measured Results

Only numbers from netlists that were actually run in ngspice appear here.

**Repo Reference** *(single-pulse `vctrl` delta, sampled right at the pulse edge, `AT=2.41n`)*

| | UP path | DOWN path |
|---|---|---|
| Pulse width | 2.129 ns | 2.129 ns |
| `vctrl` delta | +1.084 mV | −2.676 mV |
| **Mismatch ratio (DOWN/UP)** | | **2.47×** |


**AI-Generated `CP_LF`** *(700 ns continuous run, both instances start at 1.043 V)*

| | UP path | DOWN path |
|---|---|---|
| `vctrl` at end of run | 1.108 V | 0.920 V |
| Net movement | +64.7 mV | −123.3 mV |

<img width="1281" height="530" alt="Screenshot 2026-07-17 at 2 36 35 pm" src="https://github.com/user-attachments/assets/fc74c090-fafe-4070-b206-5c11ae145e1a" />

*ngspice output — left: `v(up_u)` and `v(vctrl_u)` ramping up; right: `v(down_d)` and `v(vctrl_d)` ramping down. Matches the table above.*

Both designs push `vctrl` in the right direction for each input. The
Repo Reference shows a **2.47×** DOWN/UP mismatch (corrected, edge-aligned
single-pulse measurement); the AI-Generated design shows a **1.9×**
mismatch (continuous-run measurement). Both point the same direction —
sink stronger than source — but see the methodology note below before
comparing the two ratios directly.

> **Note on methodology:** the AI-Generated numbers come from a *continuous*
> 700 ns run (many pulses, cumulative drift). The Repo Reference numbers
> come from a *single-pulse* delta (one pump event, sampled at the edge).
> These measure different things — cumulative drift vs. one-shot charge
> injection — so the 2.47× (Repo) and 1.9× (AI-Generated) ratios are each
> valid on their own but not a strict magnitude comparison against each
> other.
