# `commodity_mst_replication.ipynb`

Applies the MST (2025) three-pass hard-threshold factor model - same
methodology as `mst_replication.ipynb` - to the thesis's own commodity-linked
asset universe (N=130: 12 futures, 115 equities, 3 currencies). This is the
**observable benchmark** that the Markov-switching (MS) extension is compared
against in Section 5 of the thesis, on the thesis's own data rather than the
original MST equity universe.

## Differences from the MST equity replication

| Aspect | MST equity (`mst_replication.ipynb`) | Commodity (this notebook) |
|---|---|---|
| Frequency | Monthly | Monthly, compounded from weekly returns |
| N | 130 | 130 (different composition: futures + equities + currencies) |
| Threshold variable | ADBear | ADBear (same series, re-estimated θ̂ on this universe) |
| Pass 3 factors | RmRf | BM, MOM, HP, BASIS |

θ̂ is re-estimated by grid search on this universe rather than reused from the
equity replication - the threshold is universe-specific, not transferable.

## Required inputs (in `Input/`)

- `test_asset_universe.csv` - weekly returns, N=130 commodity-linked portfolios.
- `ADBear_Monthly_Formation.csv` - monthly ADBear state variable.
- `mExcess_Returns_Farago_Tedongap_201508_LM.csv` - used only to recover the correct monthly date index for ADBear (same file as the MST equity replication).
- `pass3_factors.csv` - weekly BM/MOM/HP/BASIS factor returns for the Pass 3 rotation.
- `bcom_daily.xlsx`, `VIXCLS.xlsx` - used only in Section 8 (robustness check), not in the main pipeline.

## Section map

| Section | Content |
|---|---|
| 0 | Imports, constants, shared helpers (identical logic to MST equity notebook), data loading and weekly→monthly compounding |
| 0.5 | ADBear descriptive statistics (pre-threshold) |
| 1 | Table 1 - grid search for θ̂ (in-sample and full-sample), regime statistics |
| 2 | Table 2 - Trapani factor-count test, % variance explained |
| 3 | Table 3 - in-sample goodness of fit, bootstrap p-values (B=2000) |
| 4 | Table 4 - whole-sample estimation |
| 5 | Table 5 - out-of-sample (L_OOS=120), OOS bootstrap |
| 6 | Pass 3 - rotates Pass 2 premia onto BM/MOM/HP/BASIS, checks sign against theory |
| 7 | Saves all table CSVs and regime dummies to `Output/` |
| 8 | Robustness: ADBear vs BCOM vs VIX as the regime variable |

## Key functions

Shared with `mst_replication.ipynb` (identical implementation, copied rather
than imported - see the duplication note in that notebook's structure):
`make_dummies`, `demean_regime`, `pca_loadings`, `pass2_gamma`,
`compute_rmspe`, `compute_r2`, `frisch_waugh_beta`, and the three bootstrap
algorithms `bootstrap_alg12`/`13`/`14`. See `README_mst_replication.md` for
what each does - repeating it here would just be the same explanation typed
twice with no new information.

Unique to this notebook:

- **`grid_search(ret, adb, T_j, label)`** - estimates θ̂ by minimising
  `RMSPE_D(θ) + RMSPE_S(θ)` over a grid of ADBear thresholds (skipping any
  threshold that would leave fewer than `T_MIN` months in either regime).
  This is what Table 1's grid search actually is - the equity replication
  hard-codes θ̂ from the paper instead of estimating it, since MST already
  published it for that universe; this commodity universe has no published
  threshold, so it has to be estimated here.
- **`trapani_test(lam, N_, T_j, cdelta, n_runs, base_seed)`** - Trapani (2018)
  randomised sequential test for the number of common factors: perturbs the
  eigenvalue sequence with simulated noise `n_runs` times and takes the median
  factor count selected, rather than a single deterministic eigenvalue cutoff.
  `cdelta` (the ε in Table 2's columns) controls the perturbation's growth
  rate - different ε values are shown side-by-side precisely because the test
  is somewhat sensitive to this choice, which is worth knowing before treating
  one ε's factor count as definitive.

## Key constants

- `THETA_HAT_IN`, `THETA_HAT_FULL` - grid-searched thresholds (in-sample and full-sample), re-estimated here, not inherited from the equity replication.
- `N_FACTORS = 6`, `T_MIN = 20`, `L_OOS = 120` - same conventions as the equity replication.
- `B_BOOT`, `B_BOOT_T4`, `B_BOOT_OOS = 2000` - bootstrap draws; set to `1` for a fast run.
- `HAS_FACTORS` - guards Pass 3; skips gracefully if `pass3_factors.csv` isn't present.

## Outputs (`Output/`)

- `commodity_mst_t3_adjr2.csv`, `commodity_mst_t3_rmspe.csv`
- `commodity_mst_t4_adjr2.csv`, `commodity_mst_t4_rmspe.csv`
- `commodity_mst_t5_adjr2.csv`, `commodity_mst_t5_rmspe.csv`
- `commodity_mst_regime_dummies.csv` - used downstream to compare against MS regime assignments.
- `fig_commodity_mst_grid_in.pdf/png`, `fig_commodity_mst_grid_full.pdf/png`, `fig_adbear_descriptive_pure.pdf`