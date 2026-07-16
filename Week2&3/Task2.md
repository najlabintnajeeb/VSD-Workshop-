# Task 2 — Phase Frequency Detector: Independent Dual-DFF Design, ngspice Verification

## What is Task 2?

Task 2 is the design and independent verification of a gate-level Phase Frequency Detector (PFD) for the PLL, built from SKY130 HD standard cells using the classic dual-D-flip-flop topology. The design was generated with AI assistance and verified end-to-end in ngspice.



---

## Block 1: Phase Frequency Detector (PFD)

Phase-Frequency Detector (PFD) — the front-end block of a PLL. It compares the edges of the reference clock (f_clk_in) against the feedback/VCO clock (f_vco) and produces two pulse outputs, up and down, whose pulse widths are proportional to the phase/frequency difference between the two clocks. These pulses later drive a charge pump + loop filter to steer the VCO.
Unlike a simple two-D-flip-flop PFD, this design is built entirely from NAND gates and inverters (no explicit DFF primitive). Each channel (reference and feedback) forms an edge-triggered latch out of cross-coupled NAND gates (X6/X7 for the top channel, X8/X9 for the bottom), and:

X10 / X3 invert the incoming clocks (f_clk_in, f_vco).
X2 / X11 are the "set" NAND gates for each channel's latch, clocked by the rising edge of each input.
X6/X7 and X8/X9 form the storage (memory) elements of each channel — functionally replacing the two D-flip-flops of a conventional PFD.
X12/X13 and X14/X15 are delay-buffer chains (two inverters = one buffer delay) that shape the timing of the NAND3 combination stage and help avoid a dead zone.
X1 (NAND4) combines internal state nodes from both channels — this is the reset generator. When both channels have triggered (i.e., both up and down conditions are met simultaneously), X1's output resets both latches, which is what keeps the PFD's dead zone minimized.
X4 (NAND3) and X5 (NAND3) merge the delayed edge, latch state, and the common reset signal from X1 to produce the raw (active-low) up/down pulses.
X16 / X17 are final inverters that convert those internal active-low pulses into the clean, active-high up and down outputs (each loaded with a small 6 fF cap, C1/C2, representing wiring/gate load).

Net result: if f_clk_in leads f_vco in phase, up pulses wider than down; if f_vco leads, down pulses wider; if they're in phase, both output very narrow (ideally zero) pulses.
### Objective

Design and verify a gate-level Phase Frequency Detector for the PLL using the SKY130 HD standard-cell library, built from two D-flip-flops and a NAND2 asynchronous reset gate. This is the primary PFD deliverable for Task 2.

### PFD Operation

The PFD compares the phase and frequency of the reference clock (`f_clk_in`) and the feedback clock (`f_vco`).

- Rising edge of `f_clk_in` sets `up`.
- Rising edge of `f_vco` sets `down`.
- When both outputs go HIGH simultaneously, the NAND2 asynchronous reset clears both flip-flops.
- The width of the UP/DOWN pulse is proportional to the phase difference between the two input clocks.

```
              +-------------+
f_clk_in ---->|  dfrtp_1    |------ up ----+
              +-------------+              |
                                            NAND2 ---- rst_b
                                            |
              +-------------+              |
f_vco  ------>|  dfrtp_1    |---- down ----+
              +-------------+
```

### AI Prompt Used


### AI-Generated SPICE Netlist

<details>
<summary><code>tb_pfd_dff.spice</code> (click to expand)</summary>

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
* plot V(f_clk_in)+6 V(f_vco)+4 V(up)+2 V(down)
.endc

*** Measurements ***
.meas tran up_pw   TRIG v(up)   VAL=0.9 RISE=1 TD=0n  TARG v(up)   VAL=0.9 FALL=1 TD=0n
.meas tran down_pw TRIG v(down) VAL=0.9 RISE=1 TD=58n TARG v(down) VAL=0.9 FALL=1 TD=58n

.end
```

</details>

---

### Simulation Results

**Status:** ✅ Simulated and verified in ngspice.

Initial operating point (t = 0): `up` and `down` both settle at ≈0 V (2.7e-9 V, effectively zero), `rst_b` at 1.8 V — flip-flops correctly reset before the first clock edge. Transient run: 10360 data rows over 100 ns.

| Measurement | Trigger time | Target time | Pulse width | Interpretation |
|---|---|---|---|---|
| `up_pw` | 2.79316e-10 s (0.279 ns) | 2.48607e-09 s (2.486 ns) | **2.20675e-09 s ≈ 2.207 ns** | UP correctly asserted in Phase 1, where `f_clk_in` leads `f_vco` by 2 ns |
| `down_pw` | 5.82803e-08 s (58.280 ns) | 6.04865e-08 s (60.486 ns) | **2.20621e-09 s ≈ 2.206 ns** | DOWN correctly asserted in Phase 2, where `f_vco` leads `f_clk_in` by 2 ns |

Both UP and DOWN paths fired cleanly within their respective test phases, with near-symmetric pulse widths (2.207 ns vs. 2.206 ns) — consistent with the 2 ns phase offset injected by the `Bvco` stimulus and confirming correct phase-to-pulse-width conversion in both directions.

---

### Comparison Against the Repo-Native PFD

The reference repo's own PFD (`pfd.cir`) uses a different implementation — a flattened, custom NAND/inverter combinational tree rather than discrete flip-flops. It was simulated earlier (`tb_pfd.spice`) as a benchmark to compare against.

| Feature | Task 2 Deliverable — Independent Dual-DFF PFD | Reference Benchmark — Repo-Native (NAND/INV tree) |
|---|---|---|
| Topology | Classic textbook dual-D-flip-flop + NAND2 reset | Flattened custom combinational NAND/inverter network |
| Standard cells | `dfrtp_1` (×2), `nand2_1` (×1) | NAND, INV gates only |
| Sequential elements | 2 discrete D flip-flops | None (purely combinational edge-detect network) |
| Source | AI-generated, independently designed for this task | Matches the actual repo netlist (`pfd.cir`) |
| Verification status | ✅ Simulated in ngspice, both `up_pw` and `down_pw` confirmed | ✅ Simulated in ngspice, only `up_pw` confirmed |
| Measured `up_pw` | 2.207 ns | 2.083 ns |
| Measured `down_pw` | **2.206 ns — successfully measured** | Out of interval — no valid DOWN pulse captured in the test window |
| Gate count | Higher (FF cells are more transistor-dense) | Lower |
| Debug difficulty | Low — standard topology, main risk was confirming `dfrtp_1`/`nand2_1` pin order | Low once topology was understood from the repo |
| Traceability to reference design | Independent — not copied from the repo | Direct — this *is* what the repo implements |

### Which Is Better, and Why This Was Used

| Question | Task 2 Deliverable — Independent Dual-DFF PFD | Reference Benchmark — Repo-Native (NAND/INV tree) |
|---|---|---|
| Which is the Task 2 deliverable? | ✅ **Yes — this is the design being built and verified for Task 2** | No — used only as a comparison benchmark |
| Why | This design's own UP and DOWN paths were both independently verified with clean, symmetric pulse widths (2.207 ns / 2.206 ns) for a 2 ns injected phase offset, giving complete, self-contained proof of correct PFD behavior in both directions | Its DOWN path never produced a measurable pulse in the tested window, so on its own it only demonstrates half of PFD operation |
| Verification completeness | Both `up_pw` and `down_pw` captured with real TRIG/TARG times | Only `up_pw` captured; `down_pw` reported out of interval |
| Design origin | Independently built to the classic PFD topology, not copied from the repo | Copied from the reference repo's actual implementation |
| Value of keeping the benchmark | — | Useful as a sanity check that the repo's own gate-level PFD is at least partially consistent with expected UP behavior, and as later context if repo-fidelity comparisons are needed |

---

### Files in This Block

- `README.md` — this document
- `tb_pfd_dff.spice` — Task 2 PFD deliverable (independent dual-DFF design), verified
- `tb_pfd_dff_out.txt` — raw `wrdata` transient output
- `waveform.png` — plot of `f_clk_in`, `f_vco`, `up`, `down`, `rst_b` from `tb_pfd_dff_out.txt`
- `pfd.spice` / `tb_pfd.spice` — reference-repo benchmark PFD, kept for comparison

---

### Conclusion

The Task 2 PFD deliverable — an independently-designed dual-D-flip-flop PFD built from SKY130 HD standard cells — was fully verified in ngspice. Both UP and DOWN pulses were correctly generated in response to the two-phase test stimulus, with measured pulse widths of 2.207 ns and 2.206 ns respectively, closely matching the injected 2 ns phase offset in each direction. This gives complete, symmetric verification of PFD behavior that the repo-native NAND/inverter benchmark could not fully demonstrate on its own, since its DOWN path never produced a measurable pulse. This dual-DFF PFD is the block carried forward into the Charge Pump + Loop Filter, VCO, Frequency Divider, and full closed-loop PLL stages of the project.
