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

## Sub-Questions, Goals, and How Lean Solves Them

The pregroup tree above is not metaphor — it is exactly the shape of
the data flow between a Lean goal and the cogitator's sub-questions.
This section makes that correspondence explicit.

### The Lean side: a goal is a typing context with an open hole

When you write `example : IsPrime 5 := by ...`, Lean records a
*goal*:

```
⊢ IsPrime 5
```

The turnstile (`⊢`) is the *typing judgement*; everything left of
it is the local context, everything right of it is the *type to
inhabit*. Each tactic is a function from goals to goals — it
either closes the goal (returns `[]`) or refines it into one or
more sub-goals.

For `IsPrime 5` written as `by decide`, the chain Lean walks is:

```
⊢ IsPrime 5
   │  unfold Decidable instance for IsPrime
   ▼
⊢ decide (IsPrime 5) = true
   │  reduce decide via Decidable.decide
   ▼
⊢ (∀ d : Fin 5, 2 ≤ d.val → 5 % d.val ≠ 0) = true
   │  unfold Fin quantifier into finite case split
   ▼
⊢ 5 % 2 ≠ 0 ∧ 5 % 3 ≠ 0 ∧ 5 % 4 ≠ 0 = true
   │  evaluate Nat.mod kernel-side
   ▼
⊢ true = true
   │  rfl
   ▼
[] (closed)
```

Each arrow is a *single rewrite step* the kernel performs. Each
intermediate goal is a complete typing context — a candidate type
that must be inhabited for the proof to type-check.

### The cogitator side: sub-questions are left adjoints of the goal

`LeastToMost.decompose(question)` returns a list of strings; the
skill treats those as left-adjoint types `gᴸ`. Each subq is a
*question whose answer is required* for the original goal to be
inhabited. `LeastToMost.solve(question, subqs)` then walks the
list, building `(subq, answer)` pairs — the answer is the right
adjoint `gᴿ`.

The correspondence is term-by-term:

| Lean goal step           | Cogitator subq / answer                  | Pregroup type |
|--------------------------|-------------------------------------------|---------------|
| `⊢ IsPrime n`            | "What is the definition of `IsPrime`?"    | `gᴸ` |
| unfold `Decidable`       | "What does `decide` reduce `IsPrime n` to?" | `gᴸ` |
| `decide ... = true`      | "How does the `Decidable` instance work?" | `gᴸ` |
| evaluate `Nat.mod`       | "How does `Nat.mod` work in Lean?"        | `gᴸ` |
| close with `rfl`         | "Why is `n % d ≠ 0` for each `d`?"        | `gᴿ` |
| `[]` (proof closed)      | (verbal synthesis)                        | `1` |

So a "correct" cogitator transcript is one whose subq + answer
sequence, read left-to-right, *traces the same series of typing
contexts the Lean kernel walks*. When that happens, the pregroup
contraction `gᴸ g gᴿ → 1` succeeds and the validator emits
`trust`.

### How Lean closes the goal in our pipeline

There are three tactics the skill currently understands as
acceptable terminators, mirrored in the `tag` column of
`parsed_steps.csv`:

| Terminator | Tag we expect | What Lean does                                  |
|-----------|----------------|--------------------------------------------------|
| `rfl`     | `close rfl`    | unify both sides up to definitional equality     |
| `decide`  | `invoke the`   | run the `Decidable` instance kernel-side         |
| `simp`/`norm_num` | `reduce the` / `evaluate the` | rewrite by hypotheses |

Any cogitator answer whose `verb` ∈ {`reduce`, `evaluate`,
`invoke`, `apply`, `unfold`, `close`, `rewrite`, `conclude`} is a
valid leaf of the pregroup tree. The rellm regex
`(reduce|evaluate|check|invoke|conclude|unfold|apply|rewrite) ARG`
is exactly the lexical-category assignment that gates this.

### Example: real parsed induction steps from `parsed_steps.csv`

These three rows are unmodified output from the notebook's
section 4 cogitator + rellm pass; they show the same goal
(`IsPrime n`) decomposed by three different checkers.

```
checker     test     step  subq                                                  answer                                                                  tag           verb     arg
─────────── ──────── ──── ────────────────────────────────────────────────────  ──────────────────────────────────────────────────────────────────────  ──────────── ──────── ─────
lean4lean   prime-5  0    What is the definition of IsPrime?                    A function that checks if a given number is prime ...                   apply a       apply    a
lean4lean   prime-5  1    What does decide reduce the proposition IsPrime n to? It reduces ... checks if n is divisible by any number from 2 to sqrt(n) reduce the    reduce   the
lean4lean   prime-5  2    How does the Decidable instance work?                 The Decidable instance uses a loop that iterates from 2 to sqrt(n) ...  invoke the    invoke   the
lean4lean   prime-5  3    What are the properties of Nat.mod that need ...?     Nat.mod needs to evaluate whether the modulus is non-negative ...       reduce the    reduce   the

nanoda      prime-3  0    What is the definition of IsPrime?                    A function that checks if a given number is prime or not ...            apply a       apply    a
nanoda      prime-3  1    How does decide reduce the proposition IsPrime n?     It uses trial division with all numbers from 2 up to the square root .. apply the     apply    the
nanoda      prime-3  2    What is the Decidable instance for IsPrime?           A propositional formula that can be reduced to IsPrime(n) is ¬(...)     reduce to     reduce   to
nanoda      prime-3  3    How does Nat.mod work in Lean?                        Nat.mod n m = (n - m) % m                                               apply to      apply    to
```

Reading column-by-column: the subq column is `gᴸ`, the answer is
`gᴿ`, and the (verb, arg) pair is the lexical category. The
sequence `apply a · reduce the · invoke the · reduce the` is one
admissible parse; the validator checks that *some such parse*
reaches a terminator that matches a real Lean tactic.

For the Fibonacci-induction half of the notebook (section 8), the
parse is more explicit because each step has a closed-form Lean
counterpart:

```
goal: fib 4 = 3
  step: apply fib_add_two to fib 4         →   fib 3 + fib 2 = 3
  step: apply fib_add_two to fib 3         →  (fib 2 + fib 1) + fib 2 = 3
  step: substitute base values fib 1 = 1,
        fib 0 = 0                          →  ((1 + 0) + 1) + (1 + 0) = 3
  step: evaluate arithmetic on Nat         →   3 = 3
  base: close by rfl                       →   []
```

Read as a pregroup string:
`(goal · stepᴿ · stepᴿ · stepᴿ · stepᴿ · baseᴿ) → 1`. The regex
`^(step\s)+base$` in section 8 of the notebook is exactly the
*acceptance condition* of this pregroup grammar: any prefix of
`step`'s followed by a single `base` reduces to the unit, and
nothing else does.

The skill's job is to compare a candidate cogitator transcript
against this grammar:

1. Tag every answer with its verb (using the rellm regex).
2. Concatenate the verbs into a string.
3. Check whether the resulting string is in the language
   `(step | unfold | apply | reduce | evaluate)* (close | rfl)`.
4. If yes → the transcript is a syntactically valid Lean proof
   trace; the checker producing it earns its score component.
5. If no → record the missing terminator and downgrade the score.

This is the operational meaning of "validate the usefulness of a
checker for proving the theorem of semantic 1-categories" — the
checker is *useful* exactly when its cogitator transcript
*parses* under the pregroup grammar above.

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
