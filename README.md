# QFF 2026 Basic track: common processor definitions

Common definitions for the Basic track of the Qiskit Fall Fest 2026 hackathon
(*Geometry-aware quantum cloud challenge*, original prompt by Steven Rayan, quanTA,
University of Saskatchewan).

Every team uses these definitions **unchanged** for the nine core cases, so that results
can be compared across teams. They fix, for each of the three processor models:

- the coupling graph (which pairs of qubits can run a two-qubit gate),
- the two designated ports (port A and port B),
- the native gate set,
- the noise model,
- the number of shots per circuit (10,000).

Layout, routing and the protection strategy remain your choice.

## What is in this folder

| File | Content |
| --- | --- |
| `qff_basic_benchmark.py` | the definitions, as one Python file you import |
| `requirements.txt` | the Python packages it needs |
| `getting_started.ipynb` | walkthrough: load the models, draw the graphs, compile a Bell circuit |
| `README.md` | this file |

## Setup

Requires Python 3.10 or later.

1. Unzip the folder.
2. In a terminal opened in the folder, install the packages:

   ```bash
   pip install -r requirements.txt
   ```

3. Open `getting_started.ipynb` in Jupyter or VS Code and run it.

To use the definitions in your own work, keep `qff_basic_benchmark.py` in the **same folder**
as your notebook or script, then `from qff_basic_benchmark import load_model`. If you see
`ModuleNotFoundError: No module named 'qff_basic_benchmark'`, the file is not in that folder.

`qiskit-ibm-runtime` must be version **0.50.0**: two of the models are fake backends shipped
in that release, and other releases can carry different calibration data. The file prints a
warning if a different version is installed.

## Quick start

```python
from qiskit import QuantumCircuit, transpile
from qff_basic_benchmark import load_model, SHOTS

m = load_model("hyperbolic")          # "2016", "contemporary" or "hyperbolic"
port_a, port_b = m.ports              # physical qubits (0, 3)

bell = QuantumCircuit(2, 2)
bell.h(0)                             # Hadamard on port A
bell.cx(0, 1)                         # CNOT from port A to port B
bell.measure([0, 1], [0, 1])          # measure only the two ports

# Compile against the model's target: logical qubit 0 -> port A, 1 -> port B.
tq = transpile(bell, m.backend, initial_layout=[port_a, port_b], seed_transpiler=1)

noisy = m.backend.run(tq, shots=SHOTS, seed_simulator=1).result().get_counts()
ideal = m.ideal_backend.run(tq, shots=SHOTS, seed_simulator=1).result().get_counts()
```

## The three processor models

| Model | Qubits | Couplers | Ports (A, B) | Native gates | Noise |
| --- | --- | --- | --- | --- | --- |
| `"2016"` | 5 | 4 | (0, 4) | cx, u1, u2, u3 | `FakeBurlingtonV2` calibration data (median CX error 1.6%, readout 3.2%) |
| `"contemporary"` | 156 | 176 | (126, 129) | cz, rz, sx, x | `FakeFez` calibration data (median CZ error 0.39%, readout 0.76%) |
| `"hyperbolic"` | 35 | 42 | (0, 3) | cz, rz, sx, x | synthetic, at the `FakeFez` medians: 1q 2.29e-4, 2q 3.90e-3, readout 7.57e-3 |

The two real-device models use the noise model Qiskit Aer builds from the fake backend's
calibration data (`AerSimulator.from_backend`): gate errors, readout errors and T1/T2.

In all three models the ports are **3 couplers apart**, so no model gets an easier task
simply because its ports are closer. Compiling the plain Bell circuit therefore needs two
SWAPs plus the CNOT (7 two-qubit gates) on every model. Differences between the models come
from their noise and from how much room the graph leaves for protection circuits (for example,
where an ancilla can be placed next to both ports).

On the two fake-backend models the ports were picked by one rule: among all qubit pairs exactly
3 couplers apart, the pair whose shortest path has a mean two-qubit-gate error closest to the
device median. This avoids both the best and the worst parts of the device.

`getting_started.ipynb` draws all three coupling graphs with the ports highlighted.

### 2016: early-era five-qubit reference

The coupling graph is the five-node reference graph of the original prompt: one hub qubit
coupled to three others, one of which is coupled to a fifth. Qiskit ships no calibration data
from 2016 itself. Several of IBM's five-qubit cloud devices (Burlington, Ourense, Vigo and
others) used exactly this graph, and the oldest such snapshot in Qiskit is `FakeBurlingtonV2`
(calibrated 11 June 2020). The model uses that device's graph, gate set (`u1, u2, u3, cx`) and
calibrated noise. In its numbering, qubit 1 is the hub and the ports are the two ends of the
path 0 - 1 - 3 - 4.

### Contemporary IBM-style

IBM Heron-style processor: `FakeFez` from `qiskit_ibm_runtime.fake_provider`, with its graph,
gate set and calibrated noise.

### Hyperbolic-inspired

A hypothetical processor whose couplers are the edges of a finite patch of the {7,3} hyperbolic
tiling. Every face is a heptagon and three faces meet at every interior vertex. The patch is
the central heptagon plus the 7 heptagons that share an edge with it: 35 qubits, 42 couplers,
14 qubits of degree 3 and 21 boundary qubits of degree 2. It is built geometrically in the
Poincaré disk (see `hyperbolic_patch` in `qff_basic_benchmark.py`).

There is no device to take calibration data from, so the noise is a synthetic circuit-level
model at today's typical gate quality: the median one-qubit, two-qubit and readout errors of
`FakeFez`. The ports are opposite vertices of the central heptagon.

## What the synthetic noise numbers mean

The hyperbolic model (and the "common" variant below) uses three numbers:

- **1q**: average gate error (1 − average gate fidelity) of every `sx` and `x` gate,
- **2q**: average gate error of every `cz` gate,
- **readout**: probability that a measured bit is flipped (the same for 0→1 and 1→0).

`rz` gates are noiseless (they are implemented in software on IBM hardware). There is no idle
or decoherence noise. Each gate error is applied as a depolarizing channel. Aer's depolarizing
parameter λ relates to the average gate error e by λ = e·d/(d − 1), with d = 2ⁿ for an n-qubit
gate; for example, a 0.39% two-qubit error gives λ = 0.0039 × 4/3 = 0.0052.

## Two noise variants

| Call | Gate set and noise | Use it for |
| --- | --- | --- |
| `load_model(name)` (same as `noise="native"`) | each model's own, as in the table above | the nine core cases |
| `load_model(name, noise="common")` | `cz, rz, sx, x` and the synthetic `FakeFez`-median noise on all three graphs | Task 6: separating the effect of the graph from gate-set and noise differences |

With `noise="common"` the three models differ only in their coupling graph.

## What a `ProcessorModel` contains

| Attribute | Meaning |
| --- | --- |
| `ports` | `(port_A, port_B)` physical qubits |
| `edges` | undirected couplers `(u, v)`, `u < v` |
| `basis_gates` | native gates the compiled circuit uses |
| `backend` | noisy `AerSimulator`; its `target` enforces the coupling graph and gate set |
| `ideal_backend` | the same target without noise, for the ideal baseline |
| `target` | compilation target, e.g. `generate_preset_pass_manager(target=m.target, ...)` |
| `noise_model` | the Aer noise model inside `backend` |
| `noise_source` | `"FakeBurlingtonV2"`, `"FakeFez"` or `"synthetic"` |
| `error_rates` | the three synthetic error rates, or `None` for fake-backend noise |
| `shots` | 10,000 shots per circuit |
| `graph()`, `port_distance()` | rustworkx graph of the couplers; shortest-path length between the ports |

`draw_model(m)` draws a model's graph with port A in green and port B in purple.
`fake_backend("2016")` and `fake_backend("contemporary")` return the underlying fake backends.

## Conventions and pitfalls

- **Bell pair:** Hadamard on port A, CNOT from port A to port B.
- **Bit order:** with `initial_layout=[port_a, port_b]` and `measure([0, 1], [0, 1])`,
  classical bit 0 is port A. Qiskit prints counts with bit 0 on the **right**, so `"01"` means
  port A = 1, port B = 0. The notebook checks this.
- **Always compile against the target.** The simulators do **not** reject a two-qubit gate
  between uncoupled qubits at run time. A circuit that skips compilation will run, but it is
  not a valid result, and submissions are checked for this.
- **Measure only what you need.** Add measurements to the small circuit before compiling.
  `measure_all()` after compiling measures every qubit of the device, including idle ones with
  readout noise.
- **Primitives:** `backend.run(...)` works directly. For the V2 primitives, use
  `qiskit.primitives.BackendSamplerV2(backend=m.backend)` or
  `BackendEstimatorV2(backend=m.backend)`; for an observable on a compiled circuit, map it with
  `observable.apply_layout(compiled.layout)`. (`qiskit_ibm_runtime.SamplerV2` is deprecated as
  of 0.50.0.)
- Report the `seed_transpiler` and `seed_simulator` values you use.

## Questions

Write to qff@iitj.ac.in with "Hackathon" in the subject line. Hackathon details:
https://events.iitj.ac.in/qff/#hackathon
