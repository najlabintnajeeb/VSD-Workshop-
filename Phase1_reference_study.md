# Phase 1 — Reference Study: `nitjsr_pll_130nm` 7-Stage Current-Starved Ring VCO

## 1. Objective
Study the reference repository's 7-stage current-starved ring VCO to understand its four-transistor current-starved inverter, bias circuit, ring connection, and output buffer — for conceptual reference only. No files (`.mag`, GDS, netlists) from the reference repository were copied, modified, or reused at any point in this work; all subsequent netlists and layouts (Phases 2–5) were independently authored.

## 2. AI Prompts Used
- Prompt to summarize the reference repository's overall VCO architecture and identify its four key building blocks (inverter, bias, ring connection, buffer)
- Prompt to interpret the reference's bias-circuit schematic netlist and explain its self-biasing mechanism
- Prompt to compare the reference's ring stage count and output-tap strategy against what would be needed for the 3-stage/5-stage variants required by this task

## 3. Reference Repository Overview
Source: `https://github.com/himansh107/nitjsr_pll_130nm`, SKY130 130nm PDK, ngspice/xschem-based PLL building blocks.

### 3.1 Current-Starved Inverter (4T topology)
A standard CMOS inverter pair (1 NMOS + 1 PMOS, gates tied to the logic input, drains tied to the logic output) augmented with two additional "starving" transistors:
- A PMOS **header**, gate driven by `Vp`, inserted between VDD and the PMOS switch's source — throttles the pull-up current
- An NMOS **footer**, gate driven by `Vn`, inserted between GND and the NMOS switch's source — throttles the pull-down current

This is the exact 4-transistor topology independently reproduced and verified in Phase 2 (`cs_inv.spice`).

### 3.2 Bias Circuit
A self-biased current generator: an NMOS (gate tied to the control voltage `Vctrl` via a small series resistor) pulls current from a shared node; a diode-connected PMOS sources current into that same node from VDD. The node self-settles to a voltage (`Vp`) that balances the two currents, while `Vn` is tied directly to `Vctrl`. As `Vctrl` rises, `Vn` rises and `Vp` falls together, coordinately reducing the starving effect on both the NMOS footer and PMOS header across every ring stage.

This concept was independently reproduced and verified in Phase 4 (`bias.spice`), with different device sizing and standard (rather than the reference's non-standard) body-tie conventions.

### 3.3 Ring Connection
7 instances of the current-starved inverter connected in a loop (output of stage *N* feeds input of stage *N+1*, with the final stage's output feeding back to the first stage's input), all sharing the same `Vp`/`Vn` bias nodes from the single bias circuit instance. An odd stage count ensures net loop inversion, satisfying the Barkhausen criterion for oscillation.

This connection pattern was independently reproduced at 3-stage and 5-stage counts (per this task's requirement, rather than the reference's 7-stage) in Phase 4 (`vco3.spice`, `vco5.spice`).

### 3.4 Output Buffer
A single NMOS/PMOS inverter pair tapping one internal ring node, isolating the ring from external loading and squaring up the tapped node's waveform into a cleaner digital output (`osc`).

This buffer structure was independently reproduced, identically in form, at the end of both `vco3.spice` and `vco5.spice`.

## 4. Scope Boundary (What Was NOT Done)
- No `.mag` layout files from the reference repository were opened, copied, or used as a geometry source at any stage.
- No netlist files (`.spice`, `.cir`) from the reference repository were included, copied, or textually reused.
- Device sizing, body-tie conventions, bias device widths, and ring stage counts were independently chosen (see Phase 2/4 comparison tables) rather than matched to the reference's specific values.
- A separate, unrelated sample `.mag` file (used purely to understand Magic's file-format syntax — section ordering, layer names, label/port conventions) was used in Phase 3 as a structural reference for the AI layout tool. This sample was not from the `nitjsr_pll_130nm` repository and was explicitly excluded from the geometry of the final, DRC/LVS-verified `cs_inv` layout's electrical design intent (see Phase 2/3 report, Section 6, entries on layout sizing correction).

## 5. Observations
- The reference's four building blocks map cleanly onto four independently-verifiable deliverables for this task: `cs_inv` (Phase 2/3), `vco_bias` (Phase 4), the ring connection pattern (Phase 4, applied at 3 and 5 stages), and the output buffer (Phase 4, embedded in both ring netlists).
- Studying the reference's actual bias-circuit netlist (rather than assuming a generic current-mirror structure) was directly useful — an initial draft bias circuit (a simpler two-transistor mirror) was superseded once the reference's self-biased, `Vctrl`-tied-to-`Vn` mechanism was understood, since it more directly matches the coordinated `Vp`/`Vn` control behavior already validated in Phase 2's delay sweep.
- This document, together with the Phase 2–5 reports, establishes a clear provenance trail: concept study (Phase 1) → isolated cell verification (Phase 2) → layout verification (Phase 3) → hierarchical circuit construction (Phase 4) → pre-layout simulation (Phase 5), satisfying the task's requirement to use the reference for understanding only.
