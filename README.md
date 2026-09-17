# Quantum Circuit Optimization with ZX-Calculus (PennyLane + PyZX)

A tutorial notebook that takes a quantum circuit written in
[PennyLane](https://pennylane.ai/), rewrites it as a **ZX-diagram** with
[PyZX](https://pyzx.readthedocs.io/), simplifies and re-optimizes it, converts
it back to a runnable circuit, and then verifies that the optimized version
produces the same output with substantially fewer gates.

Written for the
[Quantum Computing course](https://www.unibo.it/it/studiare/insegnamenti-competenze-trasversali-moocs/insegnamenti/insegnamento/2025/535208/orariolezioni)
at the University of Bologna.

## The experiment

**What ZX-calculus is.** A ZX-diagram represents a quantum computation as a
graph of green (Z) and red (X) *spiders* joined by wires, rather than as a
fixed sequence of gates on fixed qubit lines. The formalism comes with a set of
sound and complete **rewrite rules** — spider fusion, identity removal, colour
change, local complementation, pivoting — that transform a diagram into an
equivalent one. Because the graph forgets the rigid circuit structure, a
simplifier can find cancellations that gate-level peephole optimisation cannot
see. The price is that an arbitrary ZX-diagram is not necessarily a circuit any
more, so a separate **extraction** step is needed to get back to something
runnable.

**The pipeline in this notebook:**

1. **Define a device and a circuit.** `default.qubit` simulators on 4 and 5
   wires, with three example circuits to choose from (see below).
2. **Draw the circuit** in the usual gate-diagram form (`qml.draw_mpl`).
3. **Convert to a ZX-diagram** — `qml.transforms.to_zx`, which hands the tape
   to PyZX as a graph (Hadamard edges are drawn in blue).
4. **Simplify** — `pyzx.simplify.full_reduce`, which applies the rewrite rules
   exhaustively. The result is a much smaller graph, but with spiders of high
   arity that no longer correspond to one- and two-qubit gates.
5. **Extract a circuit** — `pyzx.extract_circuit` turns the reduced graph back
   into a genuine circuit (spiders now have at most 2 inputs / 2 outputs).
6. **Optimize** — `pyzx.optimize.full_optimize`, a circuit-level pass
   specialised on Clifford+T.
7. **Convert back to PennyLane** — `qml.transforms.from_zx`, wrap the tape in a
   `QuantumScript`, and execute it on the same device.
8. **Verify and compare.** Print the output distribution of both versions and
   compare gate counts and depth via `qml.specs`.

**Result recorded in the notebook** (for `quantum_circuit_5_random`, a
Toffoli-ladder-style Clifford+T circuit on 5 qubits):

| | gates | depth | T / T† | CNOT |
| --- | --- | --- | --- | --- |
| original | 63 | 48 | 16 / 12 | 28 |
| after `full_reduce` → `extract_circuit` → `full_optimize` | 37 | 30 | 6 / 2 | 19 |

Both versions output `|00001⟩` with probability ≈ 1 (everything else ~1e-32),
so the rewriting is verified to be semantics-preserving on this input. The
T-count in particular drops from 28 to 8 — which is the metric that matters
most for fault-tolerant implementations, where T gates are the expensive ones.

The notebook's closing note is worth keeping in mind: **the transformation does
not always reduce depth and gate count.** For an already-compact circuit the
round trip through extraction can come back *longer* than it started.

## Repository contents

`zx-calculus-pennylane.ipynb` — 55 cells, following the pipeline above.

**The three example circuits**, all measured with `qml.probs`:

| Function | Wires | What it is |
| --- | --- | --- |
| `quantum_circuit_4` | 4 | A short Clifford circuit (H, S, CNOT ladder) |
| `quantum_circuit_5_max_entaglment` | 5 | Hadamard + CNOT chain — the 5-qubit GHZ state |
| `quantum_circuit_5_random` | 5 | A deep Clifford+T circuit (multi-controlled-X decomposed into CNOT/T/T†) — the one the notebook is set to run |

To try a different one, swap the function name in the "visualize" and `to_zx`
cells and re-run from there.

**Two practical details the notebook handles explicitly:**

- PyZX closes its matplotlib figure after drawing, so every drawing cell
  re-attaches the figure to a fresh canvas manager before `plt.show()`.
- Circuit extraction may **permute the qubits**. That is why the optimized
  script measures `qml.probs(wires=[1,2,4,3,0])` rather than `[0,1,2,3,4]` —
  the permutation is undone in the measurement so the two distributions can be
  compared directly. If you change the circuit, expect to re-derive this order.

Helper functions `get_result()` and `print_dict_rows()` label the probability
vector with zero-padded bitstrings so the two runs can be read side by side.

## Running it

**Jupyter / Colab** — run the cells top to bottom; the first two cells install
the dependencies.

**Locally:**

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook zx-calculus-pennylane.ipynb
```

Requires Python 3.10+. Everything runs on PennyLane's `default.qubit` state
simulator — no quantum hardware or account needed. The drawing cells use wide
figures (`figsize=(32,8)`) because the reduced diagrams are long; widen your
browser or lower the figure width if they render small.

## References

- B. Coecke and R. Duncan, *Interacting Quantum Observables: Categorical
  Algebra and Diagrammatics*, New J. Phys. 13 (2011) — the origin of
  ZX-calculus.
- A. Kissinger and J. van de Wetering, *PyZX: Large Scale Automated Diagrammatic
  Reasoning*, EPTCS 318 (2020) — [arXiv:1904.04735](https://arxiv.org/abs/1904.04735)
- R. Duncan, A. Kissinger, S. Perdrix, J. van de Wetering, *Graph-theoretic
  Simplification of Quantum Circuits with the ZX-calculus*, Quantum 4, 279
  (2020) — the `full_reduce` / circuit-extraction algorithms.
- J. van de Wetering, *ZX-calculus for the working quantum computer scientist* —
  [arXiv:2012.13966](https://arxiv.org/abs/2012.13966)
- PyZX documentation: <https://pyzx.readthedocs.io/>
- PennyLane ZX-calculus transforms: <https://docs.pennylane.ai/en/stable/code/api/pennylane.transforms.to_zx.html>
