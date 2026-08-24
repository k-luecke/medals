# Selkov + Goldilocks — verified RTL

Two SystemVerilog notebooks pairing a fixed-point nonlinear dynamical system
with Goldilocks-field arithmetic, simulated end to end with **Icarus Verilog**
(plus **cocotb** in one, and **Yosys** synthesis in the other).

| Notebook | What it builds | How it is checked |
|---|---|---|
| [`selkov_goldilocks_rtl.ipynb`](selkov_goldilocks_rtl.ipynb) | Two independent cores: a Q8.24 Selkov integrator and a divider-free Goldilocks reducer | Differential: RTL vs a float integration of the same ODEs, and fast reduction vs the `%` operator bit-for-bit |
| [`dual_core_convergence_zk.ipynb`](dual_core_convergence_zk.ipynb) | The two wired into a pipeline: the integrator settles, then an R1CS core emits a witness over the settled state | cocotb testbench asserting convergence against the analytic fixed point |

Read `selkov_goldilocks_rtl.ipynb` first — it establishes both cores
independently and is explicit about what it does *not* claim.

---

## Why two parameter sets

Both notebooks integrate the same Selkov system

```
dx/dt = −x + a·y + x²y
dy/dt =  b − a·y − x²y
```

but choose different parameters, and the difference is the point. At the fixed
point `x* = b`, `y* = b/(a + b²)`, the Jacobian trace decides the behavior:

| a | b | tr(J) | behavior | used by |
|---|---|---|---|---|
| 0.08 | 0.60 | **+0.196** | limit cycle | `selkov_goldilocks_rtl` |
| 0.50 | 0.70 | **−1.000** | decays to fixed point | `dual_core_convergence_zk` |

The pipeline notebook needs a *settled* state to prove, so it deliberately picks
the decaying pair. The RTL notebook picks the oscillating pair to exercise the
integrator across a full orbit.

---

## The differential testing

Neither notebook asserts against a recorded golden trace. Each core is checked
against an independent implementation of the same claim:

- **Selkov core** — the Q8.24 fixed-point output is compared against a float
  forward-Euler integration of the same equations. If the RTL and the spec
  disagree, the check fails; a recorded trace would have hidden that.
- **Goldilocks reducer** — the fast path (split the 128-bit product into 32-bit
  limbs, exploit `2⁶⁴ ≡ 2³² − 1` and `2⁹⁶ ≡ −1 (mod p)`, one correction step, no
  divider) is compared bit-for-bit against a naive `x % P` module on random and
  edge vectors. The naive version simulates but does not synthesize sensibly —
  which is the entire reason this prime is used.

`` `default_nettype none `` is set in both, so an implicit wire is a compile
error rather than a silent bug. Arithmetic saturates and raises a sticky fault
flag: silent wraparound is treated as a fault, not a mode, because a wrapped
accumulator that keeps running is indistinguishable from a correct one
downstream.

---

## The original pipeline notebook

The point is the handoff. Core 1 is allowed to be approximate and iterative;
Core 2 emits a statement about Core 1's final state that can be checked without
re-running the integration.

### The two cores

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

### Layout

The notebook is self-contained and writes its own sources at run time:

| Artifact | Written by | Purpose |
|---|---|---|
| `agent_core.sv` | cell 3 | Both cores, `default_nettype none` so implicit wires are compile errors |
| `test_zk_pipeline.py` | cell 5 | cocotb 2.x testbench — settles the executor, then proves |
| `Makefile` | cell 7 | Icarus target, `-g2012` for SystemVerilog typed functions |
| `zk_pipeline_data.csv` | simulation | Per-tick `x`, `y`, `zk_ready`, `zk_witness` |

### Running it

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

### Verification notes

- The testbench asserts convergence against the analytically derived fixed
  point, not against a recorded golden trace — so the test still means something
  if the integration step changes.
- `SETTLE_TICKS = 6000` is a deliberate margin over the observed settling time,
  not a tuned minimum.
- The witness is checked against a Python reimplementation of the field
  arithmetic, so an RTL-side and a software-side disagreement fails the run.
