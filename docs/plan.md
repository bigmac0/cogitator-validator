# Plan for cogitator-validator

A skill plugin that promotes the
[`prime_kernel_homology.ipynb`](prime_kernel_homology.ipynb) pipeline
into a reusable decision process: given a Lean inductive
goal (`IsPrime n`, `fib k = n`, 1-category consistency), produce a
ranking of the eleven kernel checkers backed by persistent homology
of the rellm-tagged cogitator reasoning steps.

## Scope and Non-Goals

### In scope
- Loading `_results/{checker}_{test}.json` (the 11 × N acceptance grid).
- Loading `_prime_homology_build/parsed_steps.csv` (rellm-tagged steps).
- Computing the reasoning Betti table β₀, β₁[, β₂] via
  `betti_qiskit.betti_classical`.
- Promoting to `betti_qiskit.betti_quantum` (QPE) when point cloud
  exceeds the threshold (see SKILL.md, "Relationship to the Betti
  Phase Estimator").
- Confirming `S_rellm_consistency.lean` still elaborates as a
  precondition for any Betti-based suggestion.
- Emitting a typed verdict (`trust` | `distrust` | `insufficient-data`)
  through the pregroup-grammar decision tree.

### Out of scope
- Generating new Lean proofs by hand (this skill is a *meta*-validator).
- Replacing any individual checker (the skill measures, never proves).
- Running cogitator without user opt-in (each call ~60 s wall-clock).
- Producing time projections beyond `p = 11` with the linear
  `t = a + b(p−2)` model (zero degrees of freedom in the current fit).

## What is Realistic

The skill can confidently make:

- **Cross-checker topological judgements.** β₀ counts distinct proof
  *methods*; the largest component is the "majority opinion."
- **Reasoning loop detection.** Long H₁ bars correspond to shared
  cycles of reasoning across (checker, prime) cells.
- **Wall-time projections inside `[2, 11]`.** Linear in the
  `IsPrime` decide step count `p − 2`.
- **Estimator routing.** Classical vs QPE Betti based on `|X|` and
  `max_dim`.

The skill cannot:

- Decide primality directly. The kernel verdicts in
  `_results/{checker}_{test}.json` are authoritative.
- Replace the 1-category consistency theorem. If
  `S_rellm_consistency.lean` fails to elaborate, the entire functor
  loses its grounding.
- Extrapolate the linear fit safely past `p = 11`.

## Implementation of `prime_kernel_homology.ipynb`

The notebook has 10 sections (CHECK 1–8 plus the cogitator and
fib-induction setup). The skill maps each notebook section to a
single internal routine. The mapping is mechanical:

| Notebook section | Skill routine | Output artifact |
|------------------|---------------|-----------------|
| §1 Arena loader | `load_arena_verdicts()` | DataFrame `df_arena` (11 × N rows) |
| §2 Acceptance matrix | `build_M()` | numpy `M ∈ {0, 1}^{11×N}` |
| §3 Cogitator cache | `load_or_run_cogitator(missing)` | `steps_cache.json` |
| §4 sbert + VR/PH | `compute_reasoning_barcode(df_steps)` | `(diagrams, eps)` |
| §5 Discrete Morse | `find_critical_cells(D, eps)` | list of critical 0/1-cells |
| §6 PID controller | `pid_temperature_loop()` | `pid_trajectory.json` |
| §7 Fibonacci induction | `add_fib_rows(df_steps)` | extended `df_steps` |
| §8 Torus reference | `torus_reference_barcode()` | cached `torus_ph.npz` |
| §9 Signal flow | `emit_signal_flow_gv()` | `signal_flow.gv` |
| §10 QPE / Betti | `route_betti(K, k)` | classical or quantum β_k |

Every routine is *idempotent* — it reads the cache before computing,
and writes back atomically.

## Decision-Tree Driver

The pregroup-grammar tree in `SKILL.md` is the public API. The
driver walks the tree in textual form:

```
prove(theorem)
  Q1 in-scope?  ─Y─►  Q2 verdict cached?  ─Y─►  Q3 csv rows present?
   │ │                  │                          │
   N N                  N                          N
   │                    │                          │
   reject               run kernel arena           run cogitator
                        │                          │
                        ▼                          ▼
                       Q3 ...                     Q4 reasoning Betti table
                                                   │
                                                   ▼
                                                  Q5 S_rellm_consistency
                                                   │
                                                   ▼
                                                  Q6 score & rank
```

Each node returns a typed value — `gᴸ` (subq), `gᴿ` (answer), or
the unit `1` (accepted leaf). The pregroup reductions
`gᴸ g → 1` and `g gᴿ → 1` are what gate progress.

## Data Contracts

### `parsed_steps.csv`
Required columns, in order: `checker, test, step, subq, answer,
tag, verb, arg`. The skill MUST refuse to operate on a CSV that is
missing any column or has a row count below 33 (the 11 × 3 minimum
for a non-degenerate reasoning Betti table).

### `_results/{checker}_{test}.json`
Required keys: `status` ∈ {accepted, declined, rejected}, `correctness`
∈ {correct, incorrect, ?}, `wall_time` (float, seconds). All other
keys are ignored.

### `S_rellm_consistency.lean`
The skill calls `lake env lean
CAT_statement/S_rellm_consistency.lean` and treats any exit code
other than 0 as a hard failure of the 1-category precondition.

## Estimator-Routing Policy (concise)

```
if |X| ≤ 64                            → betti_classical (dense)
elif |X| ≤ 256                         → betti_classical (sparse)
elif |X| ≤ 1024 or max_dim ≥ 3         → betti_quantum (Aer)
else                                   → betti_quantum (IBM hardware)
```

Add `or Δₖ density < 5%` as a sufficient condition to promote
across any tier — sparse Δₖ is exactly where the QPE polynomial
speedup is honest.

## Phases of Implementation

### Phase 1 — Read-only validator (current target)
Load existing artifacts (`parsed_steps.csv`, `_results/*.json`,
`pid_trajectory.json`, `torus_ph.npz`), walk the decision tree,
emit a verdict. No new cogitator calls, no new Lean elaboration.

### Phase 2 — Cogitator-on-demand
Allow the user to opt in to filling missing `(checker, test)`
cells. Use the cached PID trajectory's last temperature.

### Phase 3 — Higher-prime expansion
Add `prime-7`, `prime-11`, `prime-13` slots. Re-fit the linear
`t = a + b(p−2)` model with proper residual analysis (now there are
degrees of freedom). Re-derive the reasoning Betti table with the
larger point cloud — this is the regime where QPE starts to pay.

### Phase 4 — Hardware estimation
Promote the Betti estimator from Aer to IBM Quantum Runtime when
`|S_k|` exceeds the simulator's practical limit. Already wired
through `betti_qiskit.py` (CHECK 6b).

### Phase 5 — Generalize to other inductive theorems
Replace `IsPrime` / `fib` with arbitrary `Decidable` propositions
whose Lean proof is `by decide`. The signal-flow graph and the
1-category consistency theorem are independent of the predicate.

## Anti-Patterns (mirror SKILL.md)

- Trusting one checker in isolation
- Extrapolating wall time past `p = 11`
- Running QPE below `|X| = 256`
- Bypassing `S_rellm_consistency.lean`
- Conflating the verdict table (`df_arena`) with the parsed-message
  table (`df_steps`)

## Cross-References

- `kims-plan.md` — sibling plan (lean-zip). Same "verified core +
  audited boundary" philosophy.
- `kims-skill.md` — sibling skill template (Zstd spec pattern).
  This plan is structured along the same lines.
- [`prime_kernel_homology.ipynb`](prime_kernel_homology.ipynb) — the implementation source (vendored in this repo).
- `S_rellm_consistency.lean` — the 1-category precondition.
- `betti_qiskit.py` — the estimator library.
- `docs/implementation-record.tex` — LaTeX record of the prior session.
