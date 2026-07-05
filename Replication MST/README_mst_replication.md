# `mst_replication.ipynb`

Python replication of Massacci, Sarno & Trapani (2025) - Tables 1–5, translated
directly from the paper's R and GAUSS/Ox replication files. This is the
**observable hard-threshold benchmark** against which the thesis's
Markov-switching (MS) extension is compared.

## Source mapping

| Notebook section | Original file                  | Output produced        |
|-------------------|--------------------------------|-------------------------|
| 0                 | -                               | Shared helpers & data   |
| 1                 | `1_Threshold_Estimation.ox`     | Table 1 (Panel A)       |
| 2                 | `Regime_Analysis_LM_Whole.csv`  | Table 1 (correlations)  |
| 3                 | `2_Selection.g`                 | Table 2                 |
| 4                 | `3_In_Sample_Analysis.R` (L=0)  | Tables 3 & 4            |
| 5                 | `4_Out_Of_Sample_Analysis.R`    | Table 5                 |
| 6                 | -                               | Saves all CSVs to `Output/` |

## Required inputs (must exist in `Input/`)

- `mExcess_Returns_Farago_Tedongap_201508_LM.csv` - N=130 excess return panel (12 commodity futures, 115 commodity-linked equities, 3 currencies), Feb 1997–Aug 2015.
- `ADBear_Monthly_Formation.csv` - monthly average ADBear series (bear-market state variable), 224 rows.
- `Regime_Analysis_LM_Whole.csv` - auxiliary series (e.g. `RmRf`) used for Table 4 diagnostics.

## Key constants

- `THETA_FULL = 0.318423` - threshold θ̂ estimated on the full sample (T=223, L=0).
- `THETA_INSAMPLE = 0.209761` (aliased to `THETA_IN` in Section 1) - θ̂ estimated on the in-sample window (T=103, L=120).
- `N_FACTORS = 6` - number of latent factors carried through Pass 1/Pass 2.
- `R_LAG = 1` - lag applied to the state variable relative to returns.
- `L_OOS = 120` - length of the out-of-sample window (Sep 2005–Aug 2015).
- `B_BOOT`, `B_BOOT_T4`, `B_BOOT_OOS` - bootstrap draw counts per table; set to `1` to skip a bootstrap (fast smoke-test run) or `2000` to reproduce the paper's reported p-values.

## Key functions

- **`make_dummies(adbear_avg, theta)`** - builds the bear/bull regime
  indicator vectors from the ADBear state variable and threshold. See the
  dummy-direction note below - this function's convention differs from the
  OOS code's, deliberately.
- **`demean_regime(X, d)`** - demeans the return panel *within* a regime: rows
  outside the regime (where `d==0`) are masked to `NaN` before averaging, so
  the regime mean is computed only over regime months, then filled back to
  zero outside the regime. This is what makes the PCA loadings in
  `pca_loadings` regime-specific rather than whole-sample.
- **`pca_loadings(X_dm, k, N)`** - static factor loadings: top-`k`
  eigenvectors of `X_dm'X_dm`, scaled by `sqrt(N)` (MST's normalisation, not
  the more common `1/sqrt(N)` - this scaling convention matters for how
  `pass2_gamma` recovers risk premia in the right units).
- **`pass2_gamma(X_ret_full, B, k, T_j)`** - Pass 2 of MST: cross-sectional
  regression of average returns on the Pass-1 loadings, **without an
  intercept** (MST's `gamma_{j,0}` is deliberately omitted, per the thesis's
  zero-intercept convention noted elsewhere in the project). Returns the risk
  premium vector.
- **`compute_rmspe` / `compute_r2`** - pricing-error summary statistics: root
  mean squared pricing error, and (adjusted) R² computed as `1 - N·RMSPE²/(R̄'R̄)`,
  with the adjustment always dividing by `N_FACTORS` (6), not by the current
  `k` - a specific MST convention worth knowing if you ever see adj-R²
  decrease as `k` increases, which looks wrong but isn't.
- **`frisch_waugh_beta(X_boot, F, T_j)`** - recovers loadings from a bootstrap
  draw via the Frisch-Waugh-Lovell partialling-out identity (demean by the
  grand mean via `M_iota`, then regress on the factor). Used inside all three
  bootstrap algorithms below, not just once, since every bootstrap draw needs
  its own re-estimated loadings.
- **`bootstrap_alg12` / `bootstrap_alg13` / `bootstrap_alg14`** - implement
  MST Appendix E.1's three bootstrap tests: `alg12` tests the unconditional
  model's pricing error against zero (both whole-sample and within each
  regime); `alg13` tests each regime's *own* conditional model against zero,
  using a mixed bootstrap (bear months resampled from the bear model, bull
  months from the bull model); `alg14` is the head-to-head comparison test
  (H₀: conditional model is no better than unconditional, in each regime).
  All three share the same wild-bootstrap-with-zero-mean-alpha structure -
  read `bootstrap_alg12` first, the other two are variations on it.

## Running

1. Place the three input CSVs in `Input/`.
2. Run all cells top to bottom - Section 0 must run first.
3. Set the `B_BOOT*` constants to `1` for a fast run without bootstrap p-values, or `2000` to reproduce the paper's reported significance levels (this is what was used for the thesis).
4. Section 6 writes all table CSVs and regime dummies to `Output/`:
   - `mst_t2_nfactors.csv`, `mst_t2_pctvar.csv`
   - `mst_t3_adjr2.csv`, `mst_t3_rmspe.csv`, `mst_t3_avgpe.csv`
   - `mst_t4_adjr2.csv`, `mst_t4_rmspe.csv`, `mst_t4_avgpe.csv`
   - `mst_t5_adjr2.csv`, `mst_t5_rmspe.csv`, `mst_t5_avgpe.csv`
   - `mst_regime_dummies.csv` - regime assignments used downstream for comparison against the MS extension.