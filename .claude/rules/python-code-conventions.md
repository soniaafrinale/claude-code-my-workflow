---
paths:
  - "scripts/**/*.py"
  - "explorations/**/*.py"
---

# Python Code Conventions

**Reproducibility is the default, not a feature.** A Python analysis script should be runnable
from a clean shell with no manual intervention and produce the same output every time.

> This rule mirrors [`r-code-conventions.md`](r-code-conventions.md) and
> [`stata-code-conventions.md`](stata-code-conventions.md) for users whose pipelines include
> Python. Forkers who mix languages in one project: all applicable rules apply on their
> respective files.

## 1. Reproducibility scaffolding

Every analysis script starts with the same shape:

```python
"""
File:    NN_descriptive_name.py
Purpose: [one-sentence description]
Inputs:  [path(s), relative to repo root]
Outputs: [path(s), relative to repo root]
Run order: Standalone | After NN_prior.py
"""

import random
import numpy as np
import pandas as pd

SEED = 20260415          # YYYYMMDD, set once, used everywhere below
random.seed(SEED)
np.random.seed(SEED)
```

- **One seed, set once, at the top** — same discipline as R's `set.seed()` (INV-9) and Stata's
  `set seed`. Never reseed inside a loop or function. If a library uses its own RNG (e.g.
  scikit-learn estimators), pass `random_state=SEED` explicitly rather than relying on the
  global seed.
- **All imports at the top.** No `import` inside functions except to avoid a genuine circular
  dependency (rare in analysis scripts).
- **All paths relative to the repository root** (INV-10's R rule, applied here too). Use
  `pathlib.Path` and build paths from a single `REPO_ROOT` / `OUT_DIR` constant, never a
  hardcoded absolute path.
- **Pin the environment.** Use a `requirements.txt` (or `pyproject.toml` + lockfile) — see
  [`/capture-environment`](../skills/capture-environment/SKILL.md), which snapshots this
  automatically for a replication package.

## 2. Numbered pipeline

Scripts live in `scripts/python/`, numbered for run order — the same shape as the R and Stata
pipelines:

```
scripts/python/
├── 00_setup.py          # pip-installed package check, paths, globals
├── 01_clean.py          # raw → cleaned panel
├── 02_descriptive.py    # summary stats, balance, attrition
├── 03_analyze.py        # main regression / estimation specs
├── 04_robustness.py     # alt specs, sensitivity
├── 05_tables_figures.py # tables (to .tex) + figures (matplotlib/seaborn)
└── 99_run_all.py        # imports and runs 01 through 05 in order
```

`99_run_all.py` is the one-command reproduction target, exactly like Stata's `99_run_all.do` —
`python scripts/python/99_run_all.py` from the repo root should produce every output the paper
cites.

## 3. Outputs convention

All outputs land in `scripts/python/_outputs/` (mirrors `scripts/R/_outputs/` and
`scripts/stata/_outputs/`):

```
scripts/python/_outputs/
├── clean_panel.parquet       # cleaned data (parquet over pickle: portable, columnar, fast)
├── descriptives.csv
├── main_results.tex          # table → .tex for direct \input{} in the paper
├── fig_eventstudy.pdf        # vector, for the paper
├── fig_eventstudy.png        # raster, for slides
└── sessionInfo.txt           # python version + `pip freeze` (or `uv pip freeze`)
```

Prefer **parquet or feather over pickle** for intermediate data — pickle is not portable across
Python/pandas versions and is a well-known reproducibility trap. Reserve pickle for objects that
genuinely can't serialize otherwise (e.g. a fitted model you need to reload, not raw data).

## 4. Tables

Never hand-format a table. Build it from the actual estimation output and write LaTeX directly,
the same `\input{}` discipline as the R/Stata conventions:

```python
import statsmodels.formula.api as smf

model = smf.ols("y ~ x1 + x2", data=df).fit(cov_type="cluster", cov_kwds={"groups": df["unit"]})

with open("scripts/python/_outputs/main_results.tex", "w") as f:
    f.write(model.summary().as_latex())  # or build a custom booktabs table — see below
```

For publication-quality tables, prefer a dedicated formatter (`stargazer`-equivalent packages
such as `linearmodels`' `compare()`, or hand-built `booktabs` via a small helper) over the raw
`.summary()` output, which is not booktabs-formatted and includes diagnostics the paper doesn't
need.

**Significance-stars convention:** `* 0.10 ** 0.05 *** 0.01` — matches the econ/AER convention
used in `stata-code-conventions.md` §5. Document it in the table note.

## 5. Estimation libraries

| Need | Library | Notes |
|---|---|---|
| OLS, GLM, basic panel | `statsmodels` | `cov_type="cluster"` for clustered SEs — check the small-sample correction matches your Stata/R baseline if replicating |
| Panel FE, IV, high-dimensional FE | `linearmodels` (`PanelOLS`, `IV2SLS`) | Closest Python analogue to `reghdfe`/`feols`; clustering df-adjustment differs from Stata by default — verify against a replication target, don't assume |
| Causal/ML-adjacent (double ML, matching) | project-specific — do not introduce without checking the causal-methods scope note below | |

## 6. Figures (matplotlib / seaborn)

- **Transparent background** for any figure destined for a Beamer slide:
  `fig.savefig(path, transparent=True, bbox_inches="tight")` — same requirement as R's INV-11.
- **Use the project palette**, matching `Preambles/header.tex` / `Quarto/theme-template.scss`
  (see `r-code-conventions.md` §4 for the current hex values) — no default matplotlib color
  cycle in a committed figure, same intent as INV-12.
- **Both vector and raster**: `.pdf` for the paper, `.png` (≥150 dpi) for slides — matches
  `stata-code-conventions.md` §8.

## 7. Numerical discipline

Mirrors [`r-code-conventions.md`](r-code-conventions.md) §8:

- **No float equality.** Never `a == b` on floats. Use `np.isclose(a, b, atol=..., rtol=...)`
  or `math.isclose`.
- **Probability clamping** before inverse-CDF calls. `scipy.stats.norm.ppf(1.0)` returns `inf`.
  Clamp to an open interval first:

  ```python
  eps = 1e-12
  p = np.clip(p, eps, 1 - eps)
  ```

- **Vectorize; don't grow lists/arrays in a loop.** Pre-allocate (`np.empty(n)`,
  `np.zeros(n)`) or build a list of scalars and convert once — never `np.append` inside a hot
  loop (it reallocates every call).
- **Explicit `dtype`.** Don't rely on pandas/numpy's inferred dtype for anything that feeds an
  estimator; a silently-inferred `object` column (mixed types) breaks `statsmodels`/`sklearn`
  in ways that are easy to miss.
- **Explicit `NaN` handling.** `pandas` silently drops or propagates `NaN` depending on the
  operation — always state whether a computation is `dropna()`-first or `NaN`-safe by
  construction; never rely on the default.
- **Deterministic bootstrap/simulation seeding.** Same rule as R: seed once at the top, and for
  nested/parallel replications, derive per-replicate seeds as `SEED + b` rather than reseeding
  from system entropy.

## 8. Common pitfalls

| Pitfall | Impact | Prevention |
|---|---|---|
| `df.merge()` without validating keys | Silent row duplication/loss on a many-to-one merge treated as one-to-one | `df.merge(..., validate="1:1")` (or `"m:1"`/`"1:m"` as appropriate) — fails loud, like Stata's `assert(3)` |
| Chained indexing (`df[df.x>0]['y'] = ...`) | `SettingWithCopyWarning`, silently doesn't modify the original | Use `.loc[row_mask, "y"] = ...` |
| `pickle` for intermediate data | Breaks across pandas/numpy version bumps | Use parquet/feather (§3) |
| Default `cov_type` in `statsmodels`/`linearmodels` | Understates SEs if the design has clustering and you didn't set it explicitly | Always pass `cov_type="cluster"` (or the design-appropriate alternative) explicitly, and note the choice in the table |
| Copy-vs-view ambiguity (`df2 = df1`) | Mutating `df2` silently mutates `df1` | `df2 = df1.copy()` whenever independent mutation is intended |
| `pd.read_csv` inferring dates/types inconsistently across runs/machines | A rerun on a different pandas version parses a column differently | Pass explicit `dtype=` and `parse_dates=` |

## 9. Causal-inference method scope

This repository does not prescribe *how* to implement a specific causal-identification strategy
(DiD, RD, IV, synthetic control, matching, event studies) until the project owner has given a
current, dated sign-off on the specific text — see
[`meta-governance.md`](meta-governance.md)'s owner ruling. This file covers coding discipline
(reproducibility, numerical correctness, table/figure conventions) only; which estimator or
identifying assumption to use is a `/review-paper` and project-design question, not a
conventions-file one.

## 10. Code quality checklist

```
[ ] Imports at top; one SEED constant set once
[ ] All paths relative to repo root (pathlib)
[ ] requirements.txt / lockfile present (or /capture-environment run)
[ ] Merges validated (validate="1:1" etc.)
[ ] Clustering / SE choice explicit and documented in table note
[ ] Figures: transparent=True, project palette, both PDF and PNG
[ ] Intermediate data as parquet/feather, not pickle
[ ] Numerical discipline: no float ==, probability clamping, vectorized (no loop-append)
[ ] sessionInfo.txt (python version + pip freeze) written to _outputs/
```

## Enforcement

- [`/audit-reproducibility`](../skills/audit-reproducibility/SKILL.md) handles Python outputs
  (parquet/CSV) alongside R `.rds` and Stata `.dta`.
- A dedicated `/review-r`-equivalent for Python is not yet built — review Python scripts with
  `/review-r`'s checklist applied by analogy, or a general code review, until one exists.

## Cross-references

- [`r-code-conventions.md`](r-code-conventions.md) — analogous discipline for R-first pipelines.
- [`stata-code-conventions.md`](stata-code-conventions.md) — analogous discipline for Stata-first pipelines.
- [`replication-protocol.md`](replication-protocol.md) — the tolerance contract that applies across R / Stata / Python.
- [`inference-robustness.md`](inference-robustness.md) — multiple-testing and specification-robustness discipline, already scoped to `.py` files.
- [`confidential-data.md`](confidential-data.md) — restricted-data handling, format-agnostic.
