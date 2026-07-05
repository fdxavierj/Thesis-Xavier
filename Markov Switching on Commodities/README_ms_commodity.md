# `ms_commodity.ipynb`

The core commodity Markov-switching (MS) results notebook: runs two full
end-to-end BLHK EM pipelines (P1: r_total=12 fixed, MST-comparable; P2:
r_total≈2, AH-selected) on the commodity asset universe, prices both against
the commodity MST hard-threshold benchmark, and runs the two formal hypothesis
tests reported in Section 5.3 of the thesis (HP premium asymmetry; paired
RMSPE comparison).

## Key functions

MST-side helpers (`mst_demean`, `mst_pca`, `mst_gamma`, `compute_rmspe`,
`compute_adjr2`) are the same logic as `mst_replication.ipynb`'s
`demean_regime`/`pca_loadings`/`pass2_gamma`/etc. under renamed functions -
see that README for what they do.

- **`newey_west_se(x, lags)`** / **`fm_tstats(X_ret, w, B, k)`** - Newey-West
  HAC standard errors for the Fama-MacBeth risk-premium estimates, with the
  lag truncation defaulting to the common `4·(T/100)^(2/9)` rule if not
  specified. `fm_tstats` weights each period's cross-sectional regression by
  `w` (either all-ones for the hard-threshold MST estimates, or the smoothed
  regime probability `xi_bear`/`xi_bull` for the MS estimates) - this is what
  lets the same function produce both the MST and MS t-statistics shown
  side-by-side in the Pass 2 output.
- **`pca_linear_rep(Xdm, r)`** - same static PCA as the MST notebooks, but
  returns eigenvalues too (needed by the AH criterion right after it).
- **`ahn_horenstein_skip1(eigvals, r_max)`** - Ahn-Horenstein (2013) growth-ratio
  factor-count criterion, "skip-first" variant matching the BM replication
  package: ratios are computed starting from the *second* eigenvalue, not the
  first, because the first eigenvalue (typically a market-wide factor) tends
  to dominate and would otherwise bias the criterion toward selecting too few
  factors. This is what sets Pipeline 2's `r_total`.
- **`blhk_filter` / `blhk_smoother` / `m_step` / `log_lik`** - the Hamilton
  filter, Kim smoother, weighted-least-squares M-step, and mixture
  log-likelihood - one full EM iteration is filter → smooth → M-step,
  repeated until `log_lik` stabilizes. (Same algorithm as `bm49_replication.ipynb`
  - see that README for the fuller walkthrough of each step, since that
  notebook's version was written first and is slightly more readable.)
- **`run_em_pipeline(X_dm, r_total, k_stars, label, rng, verbose)`** - wraps
  the EM loop in a **multi-start search**: tries 20 different initial
  regime-persistence values (`omega_grid` - the first 10 are hand-picked to
  span plausible persistence combinations, the remaining 10 are random),
  runs EM to convergence from each, and keeps whichever converged to the
  highest log-likelihood. This exists because EM is only guaranteed to find a
  *local* optimum - a single starting point can converge to a spurious
  low-persistence regime split.
- **`pipeline_pass2` / `pipeline_pass3`** - Pass 2 (risk premia + NW
  t-statistics, weighted by smoothed regime probability rather than a hard
  0/1 split) and Pass 3 (rotates the latent premia onto the observable
  BM/MOM/HP/BASIS factors) for one EM fit. Called once per pipeline (P1, P2).
- **`hp_bootstrap_mst` / `hp_bootstrap_ms`** (cell 62) - the formal one-sided
  bootstrap test for HP premium asymmetry (H₀: γ_HP,D = γ_HP,S). Both
  re-estimate Pass 2 loadings and Pass 3 rotation on each bootstrap draw under
  the null; `hp_bootstrap_ms` additionally holds the regime assignment `xi`
  fixed from the original EM fit (only the return panels are resampled, not
  the regime classification itself) - this is the "why held fixed" detail
  noted in the project's methodology notes.

## Structure

| Cells | Content |
|---|---|
| 1–5 | Config, shared MST helpers, data load, MST Pass 1/2 (hard-threshold benchmark), unconditional-fit companion |
| 7 | Shared: PCA linear representation + AH factor-count selection (sets `R_TOTAL_P1=12`, `R_TOTAL_P2`≈2) |
| 9–10 | Shared: BLHK filter/smoother/M-step/log-likelihood, `run_em_pipeline`/`pipeline_pass2`/`pipeline_pass3` |
| 12–24 | **Pipeline 1** (r_total=12, k=6): EM fit, BM diagnostics, MST concordance, Pass 2/3, pricing table, regime summary |
| 26–38 | **Pipeline 2** (r_total≈2, AH-selected k): same sequence |
| 40 | Cross-pipeline degree-of-concordance (DoC) |
| 42–47 | Figures: ADBear/regime timeline, 2×2 cross-sectional fit, MST-vs-unconditional scatter |
| 49–59 | Episode dating, crisis-coverage figure, regime-probability timeline, Pass 2 fit scatter, NW t-stat summary, Pass 3 premia summary |
| 62 | HP premium asymmetry bootstrap test |
| 63 | Paired RMSPE bootstrap test  |
| 64 | Saves smoothed regime probabilities for both pipelines |

## Key outputs

- Figures under `Output/`: `fig_cs_mst_bear/bull.png/pdf`, `fig_cs_ms_bear/bull.png/pdf`, `fig_mst_cond_vs_unc_bear/bull.png/pdf`, plus the timeline and crisis-coverage figures.
- `Output/` smoothed regime probabilities (both pipelines) via cell 64 - the only `.to_csv` call in the entire notebook.

## Key constants

- `THETA_HAT = 0.1250` - commodity MST full-sample threshold (Table 4), fixed here, not re-estimated.
- `R_TOTAL_P1 = 12`, `K_P1 = 6` - MST-comparable pipeline.
- `R_TOTAL_P2` - AH-selected, `K_P2 = R_TOTAL_P2 // 2`.
- `EM_MAX_ITER=500`, `EM_TOL=1e-7`, 20-point multi-start `omega_grid` for regime-persistence initialization.
- `B_BOOT_HP = 2000` (HP asymmetry test), `B_BOOT_PAIRED = 499` (paired RMSPE test).
