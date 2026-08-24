# Dual-Core Agent — Selkov Convergence → Goldilocks ZK Proof

A two-core hardware pipeline in SystemVerilog, simulated end-to-end with
**cocotb** and **Icarus Verilog**: one core settles a nonlinear dynamical
system, the second proves the settled state with an R1CS witness over the
Goldilocks field.

The point is the handoff. Core 1 is allowed to be approximate and iterative;
Core 2 emits a statement about Core 1's final state that can be checked without
re-running the integration.

## The two cores

**Core 1 — `selkov` (the Executor).**
A fixed-point (Q8.24) Selkov glycolytic oscillator integrated with forward
Euler. With `a = 0.5`, `b = 0.7` the Jacobian trace at the fixed point is
`-1.0`, so the state converges rather than oscillates:

```
x* = b            = 0.7
y* = b/(a + b²)   ≈ 0.707
```

All arithmetic saturates and raises a sticky fault flag. Silent wraparound is
treated as a fault, not as a mode — a wrapped accumulator that keeps running is
indistinguishable from a correct one downstream, so the design refuses to let
that happen quietly.

**Core 2 — `zk_goldilocks_r1cs` (the Prover).**
Once the executor settles, this core builds an R1CS witness over the Goldilocks
prime `p = 2⁶⁴ − 2³² + 1` and raises `zk_ready`. Goldilocks is chosen for the
usual reason: reduction mod `2⁶⁴ − 2³² + 1` is a few adds and shifts rather
than a general modular reduction, which is what makes it tractable in RTL.

## Layout

The notebook is self-contained and writes its own sources at run time:

| Artifact | Written by | Purpose |
|---|---|---|
| `agent_core.sv` | cell 3 | Both cores, `default_nettype none` so implicit wires are compile errors |
| `test_zk_pipeline.py` | cell 5 | cocotb 2.x testbench — settles the executor, then proves |
| `Makefile` | cell 7 | Icarus target, `-g2012` for SystemVerilog typed functions |
| `zk_pipeline_data.csv` | simulation | Per-tick `x`, `y`, `zk_ready`, `zk_witness` |

## Running it

Requires `iverilog`, `cocotb` 2.x, `pandas`, and `matplotlib`.

```bash
jupyter notebook dual_core_convergence_zk.ipynb
```

Run the cells in order. Cell 8 executes `make SIM=icarus` and fails loudly if
the simulation fails — it parses `results.xml` directly rather than trusting
`make`'s exit code, because cocotb records assertion failures there even when
`make` returns 0.

Cell 10 plots the executor settling with the proof event marked at the tick
where `zk_ready` first asserts.

## Verification notes

- The testbench asserts convergence against the analytically derived fixed
  point, not against a recorded golden trace — so the test still means something
  if the integration step changes.
- `SETTLE_TICKS = 6000` is a deliberate margin over the observed settling time,
  not a tuned minimum.
- The witness is checked against a Python reimplementation of the field
  arithmetic, so an RTL-side and a software-side disagreement fails the run.
