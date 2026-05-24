---
name: cogitator-validator
description: Use when validating Lean kernel checkers against a topological model of cogitator reasoning steps, suggesting which of the eleven checkers to trust for an `IsPrime`/`Fibonacci`/1-category goal, or interpreting `parsed_steps.csv` as a persistence diagram. Drives a pregroup-grammar decision tree, the rellm → sbert → VR → PH signal flow, and the QPE Betti estimator for higher-dimensional reasoning point clouds.
allowed-tools: Read, Edit, Write, Bash, Glob, Grep
---

# Cogitator Decision-Tree Validator

A spec for turning the `prime_kernel_homology.ipynb` pipeline into a
reusable validator. It manages the decision process for proving
inductive goals in Lean (`IsPrime n`, `fib k = n`, monad / 1-category
consistency) and assigns a *usefulness score* to each of the eleven
checkers:

```
always-accept, always-decline, always-reject, lean4lean, mini,
nanobruijn, nanoda, official, parse-only, sonanoda, still-nanoda
```

The verdict comes from a persistence diagram of the rellm-tagged
cogitator steps, *not* from individual kernel output, so the eleven
checkers are scored relative to each other and to a reference
topology (the 3-torus barcode).

## Inputs

| Input | Where | What it is |
|-------|-------|-----------|
| `parsed_steps.csv` | `~/lean/_prime_homology_build/parsed_steps.csv` | rellm-tagged cogitator steps; columns `checker,test,step,subq,answer,tag,verb,arg` |
| `{checker}_{test}.json` | `~/lean/lean-kernel-arena/_results/` | 11 × N JSON kernel verdicts (`status`, `correctness`, `wall_time`) |
| `S_rellm_consistency.lean` | `~/lean/LeanCat/CAT_statement/S_rellm_consistency.lean` | 1-category consistency theorem (elaborates iff the signal flow is sound) |
| `signal_flow.gv` | `~/lean/_prime_homology_build/signal_flow.gv` | Graphviz source for the rellm-monad → barcode pipeline |
| `betti_qiskit.py` | `~/lean/betti_qiskit.py` | classical + QPE Betti estimator |

## Decision Tree (pregroup grammar form)

Every validation request reduces to *one path* through this tree.
Each internal node is a goal `g`; each subq is `gᴸ`; each answer is
`gᴿ`. Acceptance is the empty string under the pregroup contraction
`gᴸ g gᴿ → 1`.

```
[ROOT]   prove(theorem)                                       :: g
   │
   ├── Q1  "is the goal in the supported class?"              :: gᴸ
   │     ├─ Y  goal ∈ {IsPrime n, fib k = n, 1-cat consistency}
   │     └─ N  return: out-of-scope; suggest manual proof
   │
   ├── Q2  "is there a kernel verdict in lean-kernel-arena?"  :: gᴸ
   │     ├─ Y  load 11 × {test} verdicts → matrix M (CHECKS 2-4)
   │     └─ N  emit cogitator subqs → rellm-tag → cache → run kernel
   │
   ├── Q3  "is there a parsed_steps.csv for these (checker,
   │       test) cells?"                                      :: gᴸ
   │     ├─ Y  sbert-embed answers → ℝ³⁸⁴ → VR(ε) → PH        :: gᴿ
   │     └─ N  invoke cogitator (~60s per missing cell)
   │
   ├── Q4  "what does the reasoning Betti table say?"         :: gᴸ
   │     β₀ → number of distinct proof methods
   │     β₁ → independent reasoning loops (target: ≥3 = T³-like)
   │     β₂ → three-way checker chambers
   │     │
   │     ├─ all β_k match torus reference → STRONG validation
   │     ├─ β₀ ≫ 1 → checkers diverge in reasoning; pick majority class
   │     └─ β₁ = 0 → no shared loop; the cell is single-shot only
   │
   ├── Q5  "does S_rellm_consistency.lean elaborate?"         :: gᴸ
   │     ├─ Y  1-category framing is sound; PH ∘ VR ∘ ι ∘ T is a functor
   │     └─ N  STOP; the signal-flow graph is not 1-categorical
   │
   └── Q6  "rank checkers by usefulness"                      :: gᴿ → 1
         score(c) =  α · (M[c, p] = OK)
                   + β · (β₀-component containing c is the largest)
                   + γ · (lifetime of c's answer in H₁ barcode)
                   − δ · wall_time(c, p) / median wall_time
         tie-break:  prefer (c, p) cells whose tag matches the verb
                     family {reduce, evaluate, unfold, apply, close}.
```

The tree terminates when every leaf either returns a checker
ranking, or rejects the goal with a typed reason. The pregroup
contraction is the *type system* of the validator: a suggestion is
emitted only if the full path reduces to `1`.

## Signal Flow Graph (what to build, what to consume)

```
Σ (rellm regex alphabet)
   │  generators
   ▼
T : X ↦ X*     (free-monoid monad on Set; CategoryTheory.Monad)
   │  η lifts atoms, μ splices traces
   ▼
cogitator + rellm steps in Σ*     ←── parsed_steps.csv lives here
   │  ι = sbert all-MiniLM-L6-v2
   ▼
X ⊂ ℝ³⁸⁴   (point cloud, |X| = #rows of parsed_steps.csv)
   │  Vietoris–Rips filtration
   ▼
K(ε) ⊂ Simplicial Sets
   │  Hₖ = combinatorial Laplacian kernel
   ▼
β₀, β₁, β₂[, β₃]   ←──  betti_qiskit.{betti_classical, betti_quantum}
   │
   ▼
barcode  ──compare──►  T³ reference (1, 3, 3, 1)
```

The `signal_flow.gv` file in `_prime_homology_build/` is the
ground-truth diagram; this skill's job is to keep that diagram in
sync with the CSV's actual contents.

## Relationship to the Betti Phase Estimator

`betti_qiskit.py` exposes:

- `clique_complex(N, edges, max_dim)` — builds K from the VR
  1-skeleton.
- `combinatorial_laplacian(K, k)` — assembles Δₖ.
- `betti_classical(K, k)` — `dim ker Δₖ` via eigendecomposition.
- `betti_quantum(K, k, n_phase_qubits, n_samples, shots)` — QPE on
  `U = exp(2πi · Δₖ / λ)` and counts phase-`0` measurements.

The skill's role is to dispatch one of these *based on size*:

| `|X|` (rows) | Recommended estimator | Rationale |
|------------:|-----------------------|-----------|
| ≤ ~64 | `betti_classical` | dense eigendecomp is sub-second |
| 64 – ~256 | `betti_classical` with sparse Δₖ | still tractable, no quantum needed |
| 256 – ~1024 | `betti_quantum` on Aer simulator | classical dense eigendecomp `O(\|S_k\|³)` starts to hurt |
| > 1024 | `betti_quantum` on IBM hardware via `QiskitRuntimeService` | only path that fits in memory |

`parsed_steps.csv` is currently 122 rows / ~60 (checker, test, step)
points. That sits firmly in the classical band. The skill should
*suggest* the classical estimator now and only emit a "promote to
QPE" recommendation when row count or maximum simplex dimension
crosses the boundary.

### Hypothetical extension to higher dimensions

If the CSV grows beyond ~256 rows (e.g., add `prime-7, prime-11,
prime-13` × all 11 checkers × ~7 cogitator subqs each, or extend to
`H₃, H₄` of the 1-skeleton), the relevant quantities are:

- `|S₀| = N` (rows)
- `|S₁| ≤ \binom{N}{2}` (pairs within ε)
- `|S₂| ≤ \binom{N}{3}` (triples within ε)
- `|Sₖ| ≤ \binom{N}{k+1}`

Dense classical β_k costs `O(\|S_k\|^3)`. QPE costs are dominated
by the unitary-exponentiation depth, which scales as
`poly(log \|S_k\|)` per shot for a sparse Δₖ. The crossover is
real around `\|S_k\| ≳ 10³`.

### Quantum advantage in this scenario

There is **no exponential speedup** for Betti numbers of the
reasoning complex at current scale — the dense linear algebra wins
on a laptop. The honest advantages of routing this through Qiskit
are:

1. **Polynomial speedup on sparse Δₖ at high `k`.** QPE measures
   `dim ker Δₖ` in time `poly(log \|S_k\|, 1/ε)` per sample versus
   `O(\|S_k\|^ω)` for classical exact eigendecomp. For Vietoris–Rips
   complexes the simplex count grows roughly as `\binom{N}{k+1}`,
   so even modest `N` makes `k ≥ 3` painful classically and
   tractable on QPE.

2. **No need to materialize Δₖ.** The QPE estimator only requires
   block-encoding access to Δₖ; the full Hodge Laplacian never
   leaves register space. Classical eigendecomp needs the dense
   matrix in RAM.

3. **Compositional with the rest of the signal-flow pipeline.**
   Because `S_rellm_consistency.lean` certifies that everything is
   a 1-categorical functor into `AddCommGrp`, the quantum estimator
   can be slotted in as a drop-in object on the right-hand side of
   the functor without re-deriving consistency.

4. **Hardware path is live.** `betti_qiskit.py` already connects to
   `QiskitRuntimeService` (CHECK 6b of the notebook), so promoting
   from simulator to real hardware is a single backend swap.

The skill therefore *recommends* but does not *require* the
quantum estimator. The threshold rule is:
`use_qpe ⇔ (|X| > 256) ∨ (max_dim ≥ 3) ∨ (Δₖ density < 5%)`.

## Suggestion Generator (what the skill emits)

For a request like *"is `nanoda` trustworthy for proving
`IsPrime 7`?"* the skill produces:

```
SUGGESTION FOR (nanoda, prime-7):
  → kernel-arena status:    [NOT YET RUN | OK | REJECT]
  → cogitator parsed-rows:  K rows in parsed_steps.csv
  → reasoning H₀ component: {checker membership of cluster}
  → reasoning H₁ bar:       birth ε₀, death ε₁  (lifetime ε₁−ε₀)
  → score:                  s ∈ [0, 1]  (formula in Q6)
  → recommended estimator:  classical (|X|<256) | QPE (else)
  → 1-cat consistency:      PASS / FAIL via S_rellm_consistency.lean
  → verdict:                trust | distrust | insufficient-data
```

For `IsPrime 7`/`IsPrime 11` (not yet in `parsed_steps.csv`) the
verdict path enters Q3-N: the skill suggests `cogitator.LeastToMost`
with `temperature` taken from the PID history file
`_prime_homology_build/pid_trajectory.json` (last value, capped at
T_HI=1.5), then re-enters at Q4.

## How to use this skill

Invoke when a user asks any of:

- "which checker should I trust for prime p?"
- "does my new checker fit the topology of the existing 11?"
- "should I run the classical or quantum Betti estimator?"
- "interpret the barcode of parsed_steps.csv"
- "is the 1-category consistency theorem still elaborating?"
- "extend the analysis to prime-7 / prime-11 / prime-13"

Do NOT invoke for:

- generic primality questions (`Nat.Prime` lemmas in Mathlib)
- unrelated Lean tactics outside the kernel-arena scope
- requests to actually run new cogitator calls without user opt-in
  (each call is ~60s of wall-clock and changes the cache)

## Anti-Patterns

- **Don't trust a single checker's verdict.** The whole point of
  the eleven-checker grid is the *cross-checker topology*. A score
  from one row is meaningless.
- **Don't extrapolate `wall_time(p)` past p = 11** with the linear
  `t = a + b(p−2)` model. Three points = zero degrees of freedom.
- **Don't run QPE below 256 points.** It is slower than the dense
  classical eigendecomp at that scale and the cache thrash is real.
- **Don't bypass `S_rellm_consistency.lean`.** If that theorem
  stops elaborating, the entire functor `PH ∘ VR ∘ ι ∘ T` loses
  its 1-categorical grounding and every Betti number is suspect.
- **Don't conflate the primary and secondary tables.** `df_arena`
  is the verdict table; `df_steps` / `parsed_steps.csv` is the
  parsed-message table. The sublevel filtration joins them but
  they are distinct objects.

## Cross-References

- `prime_kernel_homology.ipynb` — the source pipeline (sections 1–10).
- `S_rellm_consistency.lean` — the 1-category consistency theorem.
- `betti_qiskit.py` — classical + QPE Betti estimator.
- `kims-plan.md` — sibling plan in the same repo (lean-zip), shares
  the "verified core + audited boundary" philosophy.
- `docs/plan.md` (this folder) — full implementation plan.
- `docs/implementation-record.tex` — LaTeX record of the prior
  synopsis session.
