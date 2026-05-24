# cogitator-validator

A Claude Code skill plugin that validates the eleven Lean kernel
checkers against a topological model of their cogitator reasoning
steps. Backed by persistent homology of an sbert-embedded point
cloud, with a Qiskit QPE estimator for higher-dimensional regimes.

![logo](docs/logo.svg)

## What it does

Given an inductive Lean goal (`IsPrime n`, `fib k = n`, or
1-category consistency), it walks a pregroup-grammar decision tree
to produce a per-checker trust ranking grounded in:

- the verdict matrix `M ∈ {0,1}^{11×N}` from
  `lean-kernel-arena/_results/`,
- the rellm-tagged cogitator step table
  `_prime_homology_build/parsed_steps.csv`,
- the Vietoris–Rips / persistent-homology pipeline (sections 4–10
  of [`docs/prime_kernel_homology.ipynb`](docs/prime_kernel_homology.ipynb)),
- the 1-category consistency theorem in
  `LeanCat/CAT_statement/S_rellm_consistency.lean`.

## Install (as a Claude plugin)

```bash
# clone this repo somewhere
git clone https://github.com/bigmac0/cogitator-validator.git
# or point Claude at the local folder via the marketplace.json
```

Claude reads `.claude-plugin/marketplace.json` and exposes the
skill under `cogitator-validator:cogitator-validator`.

## Layout

```
cogitator-validator/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   └── cogitator-validator/
│       └── SKILL.md          ← spec / decision tree / estimator routing
├── docs/
│   ├── plan.md                       ← implementation plan
│   ├── prime_kernel_homology.ipynb   ← source pipeline (sections 1–10)
│   ├── implementation-record.tex
│   └── logo.svg
├── README.md                 ← this file
└── LICENSE
```

## Author

Ivan Rojas (impetus1@gmx.com)

## License

MIT
