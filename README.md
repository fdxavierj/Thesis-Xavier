# Bachelor Thesis BSc2 Econometrics and Economics - Bear Market Risk in Commodity-Linked Assets: A Regime-Dependent Latent Factor Approach with Markov-Switching Models

Master guide to the six folders in this repository: what each one is, what it
needs, what it produces, and the order to run things in. Run order follows
**data dependencies**, not the alphabetical order the folders happen to be
listed in.

**Before running anything:** each folder's `Input/` should already contain
the CSVs produced by whichever upstream folder(s) it depends on (right now, 
these have already been copied over). If a notebook errors on a missing file,
check the dependency table below before assuming the code is broken - most 
"file not found" errors in this package mean an upstream notebook hasn't 
been run yet, or its output landed in the wrong `Input/`.

---

## Run order

```
1. Data Collection
       │
       ├──► 2a. Replication MST on Commodities
       │
       └──► 3a. Markov Switching on Commodities ──► 4. Sector-Sorted
                                                       (D10c cross-check only)

2b. Replication MST            (independent - own equity source data)
3b. Markov Switching on MST    (independent - own equity source data)
```

- **Data Collection** must run first - everything with "on Commodities" in
  its name depends on its output.
- **Replication MST** and **Markov Switching on MST** use the standard MST
  equity universe directly (Farago-Tedongap data) and don't need Data
  Collection's output. They can run any time, independently of everything else.
- **Sector-Sorted** builds its own alternate universe internally (it has its
  own equivalent of Data Collection built in) and only reaches back into
  **Markov Switching on Commodities** for one specific check - a cross-universe
  regime-concordance comparison late in the robustness notebook. If you skip
  that one comparison, Sector-Sorted doesn't need anything from folders 1–3.

---

## Folder-by-folder

### 1. Data Collection

Builds the N=130 commodity-linked test asset universe used by every
"on Commodities" folder downstream: 12 futures tercile portfolios, 115
characteristic-sorted equity portfolios, 3 currency tercile portfolios,
combined into one weekly excess-return panel, plus the four observable Pass 3
factors (BM, MOM, HP, BASIS).

**Needs:** Datastream Excel exports (futures, currencies), CFTC COT data,
WRDS/CRSP access for the equity sorts, Fama-French factors (auto-downloaded).

**Produces (→ becomes downstream `Input/`):**
- `test_asset_universe.csv` - the N=130 panel
- `pass3_factors.csv` - the 4 observable factors
- `commodity_returns_weekly.csv` - individual futures contract returns (needed for the currency beta sort, run this before the currency notebook if running from scratch)

---

### 2a. Replication MST on Commodities

Applies the Massacci-Sarno-Trapani (2025) hard-threshold three-pass model to
the Data Collection universe. This is the observable benchmark that Markov
Switching on Commodities is compared against.

**Needs:** `test_asset_universe.csv`, `pass3_factors.csv` (from Data Collection).

**Produces:** Tables 1–5 equivalents on this universe, regime dummies, θ̂
(re-estimated on this universe - not transferable from the equity version).

---

### 2b. Replication MST

Same MST methodology, applied to the **original MST equity universe**
(N=130 equities, the paper's own asset universe), not the commodity universe.
This is the validation exercise confirming the code reproduces MST's published
numbers before it's trusted anywhere else - including on the commodity data
in 2a.

**Needs:** The standard MST equity source files (excess returns, ADBear).
**Does not need** anything from Data Collection.

**Produces:** Tables 1–5 on the original equity universe, θ̂ for that universe.

---

### 3a. Markov Switching on Commodities

The core empirical result: Barigozzi-Massacci (2025) latent Markov-switching
EM applied to the Data Collection universe (two pipelines - MST-comparable
r_total=12, and AH-selected r_total). Priced against the 2a benchmark. Runs
the two formal hypothesis tests (HP premium asymmetry, paired RMSPE
bootstrap).

**Needs:** `test_asset_universe.csv`, `pass3_factors.csv` (Data Collection).

**Produces:** Smoothed regime probabilities, pricing tables, `commodity_xi_bear.csv`
(P1's smoothed bear probability - **this is what Sector-Sorted's cross-universe
DoC check reads**).

---

### 3b. Markov Switching on MST

Same EM methodology, applied to the original MST equity universe (parallel
to 2b, not 2a). Independent of Data Collection.

**Needs:** Standard MST equity source files.

**Produces:** Equity-universe smoothed probabilities, pricing tables, and its
own paired RMSPE bootstrap result - **check whether this result has been
reconciled with the stale-vs-current-source discrepancy** flagged in that
notebook's README before citing it.

---

### 4. Sector-Sorted

**Status: robustness-only, excluded from the thesis's reported results.**
Builds an entirely separate, smaller test universe (N=52: 28 individual
futures, 12 equity sectors, 12 currencies - no characteristic sorting) as a
structural robustness check, then re-runs both MST and MS on it.

**Needs:** Its own raw Datastream/WRDS inputs (self-contained build step, not
Data Collection's output). For the one cross-universe comparison in its
robustness notebook (Section 10), it also needs `commodity_xi_bear.csv` from
**3a**.

**Produces:** A parallel N=52 universe and pricing comparison, plus the
diagnostic (D10) that's the actual reason this universe was excluded: the
regime probabilities collapse to near-binary (98% of months outside the
interior 0.05–0.95 band), meaning the headline pricing improvement here can't
be trusted as genuine regime-dependent evidence rather than a near-deterministic
split. This is documented in depth in that folder's own README - read it
before using any number from this folder in anything other than a robustness
appendix.

---

## Where each README lives

Every folder has its own README with fuller detail on the notebooks (key functions, 
exact constants, required inputs/outputs).
