# `ms_equity_final_temp.ipynb`

The equity-universe counterpart to `ms_commodit_temp_2.ipynb`: two BLHK EM
pipelines (P1: r_total=12, MST-comparable; P2: AH-selected) run on the MST
equity test-asset universe, priced against the MST equity hard-threshold
benchmark.

## Key functions

Nearly identical function set to `ms_commodit_temp_2.ipynb` - full
explanations are in that README, not repeated here in full. The one
substantive difference:

- **`pipeline_pass3_equity`** replaces the commodity notebook's
  `pipeline_pass3`: it rotates the latent Pass 2 premia onto a single
  observable factor (RmRf) instead of four (BM/MOM/HP/BASIS), since that's all
  this universe has. Same rotation math, just `k_obs=1` instead of `4` - if
  you're comparing the two notebooks side by side, don't expect the function
  signatures to line up exactly.
- No `hp_bootstrap_mst`/`hp_bootstrap_ms` equivalent exists here - there's no
  HP factor in this universe's Pass 3, so the HP-asymmetry test the commodity
  notebook runs doesn't apply. The only formal test here is the paired RMSPE
  bootstrap.

## Structure

| Cells | Content |
|---|---|
| 1–5 | Config, shared MST helpers, data load, MST Pass 1/2, unconditional-fit companion |
| 7 | Shared: PCA linear representation + AH factor selection |
| 9–10 | Shared: BLHK filter/smoother/M-step, `run_em_pipeline` (now with `verbose`), `pipeline_pass2`/`pipeline_pass3_equity` |
| 12–24 | **Pipeline 1** (r_total=12, k=6): full sequence |
| 26–38 | **Pipeline 2** (AH-selected): full sequence |
| 40 | Cross-pipeline DoC |
| 42–52 | Figures: crisis episode dating, Pass 2 fit scatter, 2×2 cross-sectional fit, MST-vs-unconditional scatter, regime-probability timeline |
| 53–54 | Paired RMSPE bootstrap test (complete, but re-run recommended - see finding #1) |
| 56 | Saves smoothed probabilities and both pipelines' pricing tables |

## Outputs (this one does save its pricing tables - contrast with the commodity notebook)

- `Output/ms_equity_smoothed_probs.csv`
- `Output/ms_equity_pricing_p1.csv`, `Output/ms_equity_pricing_p2.csv`

This is worth noting explicitly: the commodity notebook's equivalent cells
build `df_p1`/`df_p2` but never save them. This file does. If the commodity
notebook was adapted from this one (or vice versa), the `.to_csv` calls for
the pricing tables got dropped somewhere in that process - this file is the
version to copy the save logic from.

## Key constants

- `THETA_HAT` - MST equity full-sample threshold (see `mst_replication.ipynb`).
- `R_TOTAL_P1 = 12`, `K_P1 = 6`; `R_TOTAL_P2`, `K_P2` - AH-selected.
- `B_BOOT_PAIRED = 499`, `RNG_SEED_PAIRED = 19780308`.
