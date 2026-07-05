# `sector_universe.ipynb`

**Status: robustness-only, excluded from the thesis.** Builds an alternative
sector-sorted commodity test asset universe (N=52) as a robustness check
against the main characteristics-sorted universe (N=130). This universe and
its results are not part of the thesis's reported findings - see
`sector_robustness_notebook.ipynb` for why.

## What this notebook does

| Cell | Block | Content |
|---|---|---|
| 0 | - | Configuration: file paths, sample window, `MIN_FIRMS_PER_SECTOR=10`, RUB winsorisation flag |
| 1 | A | 28 individual commodity futures (no sorting), weekly log-compounded then aggregated to monthly |
| 2 | C | 12 individual commodity currencies (no beta sorting), RUB winsorised at [0.5%, 99.5%] for the 1998 devaluation outlier |
| 3 | B | 12 value-weighted equity sector portfolios via WRDS/CRSP, min 10 firms per sector-month (5 for the thin `EQ_P_AgForestry` sector) |
| 4 | - | Merge all three blocks, align to ADBear, save |

## Required inputs

- `Input/commodity_futures_data.xlsx`, `Input/commodity_currencies_data.xlsx`
  - same Datastream export format as the characteristics-universe notebooks.
- `Input/ADBear_Monthly_Formation.csv`, `Input/mExcess_Returns_Farago_Tedongap_201508_LM.csv`
  - for aligning ADBear to this notebook's own monthly calendar.
- WRDS access for Block B (Cell 3) - skippable if `Output/block_B_equity_monthly.csv`
  already exists.

## Outputs (`Output/`)

- `block_A_futures_monthly.csv`, `block_B_equity_monthly.csv`,
  `block_B_firmcounts.csv`, `block_C_currencies_monthly.csv` - intermediate,
  per-block.
- `sector_test_asset_universe.csv` - the final N=52 merged panel, **consumed
  by `sector_robustness_notebook.ipynb`**.
- `sector_adbear_monthly.csv`, `sector_block_map.csv`.

## Why this universe exists and why it's excluded

Built to test whether the thesis's main findings (regime-dependent pricing
under MS vs MST) survive a completely different asset universe construction
- individual futures/currencies instead of characteristic-sorted portfolios,
and equity sectors instead of firm-level sorts. See
`sector_robustness_notebook.ipynb` for the diagnostic that explains
the exclusion (near-binary regime probability collapse, not a data or code
bug in this build notebook).


# `sector_robustness.ipynb`

**Status: robustness-only, excluded from the thesis.** Applies both MST and
MS frameworks to the sector-sorted universe (N=52, from
`sector_universe_build.ipynb`) and runs ten labeled diagnostics (D1–D10)
investigating whether this alternative universe supports the same
regime-dependent pricing conclusion as the main N=130 characteristics
universe.

## Section map

| Section | Diagnostics | Content |
|---|---|---|
| 0 | - | Imports, shared BLHK/MST helpers (same logic as `ms_commodit_temp_2.ipynb`), data load |
| 1 | D5, D9 | AH factor-count selection (→ r_total=2); near-collinear asset pairs (WTI/Brent, Wheat CBOT/KCBT) |
| 2 | - | MST hard-threshold estimation at k=6, θ=0.125 (inherited from characteristics universe, not re-estimated here) |
| 3 | D1, D2, D7 | Within-block residual correlation; per-block zero-intercept test (currency block fails, as expected); RMSPE vs idiosyncratic vol (thin-sector check) |
| 4 | - | MS EM estimation, 20-start multi-start, frequency-based regime labeling |
| 5 | D6, D4, D1(post-EM), D3 | Triple-criterion labeling agreement; Kish effective sample size; Pass-3 internal-rotation warning (factors built from the *same* futures being priced) |
| 6 | D10a, D10b | Probability diffuseness - see "the actual finding" above |
| 7 | D5b | r_total sensitivity grid (2, 4, 6) |
| 8 | D8 | RUB sensitivity - incomplete, see above |
| 9 | - | Headline pricing comparison, MST vs MS, k=1..6 |
| 10 | D10c | Homoskedastic + AH-r_total=2 combined; extended pricing table; cross-universe DoC|

## Required inputs

- `Output/sector_test_asset_universe.csv`, `Output/sector_adbear_monthly.csv`,
  `Output/sector_block_map.csv` - all from `sector_universe_build.ipynb`.
- `Output/commodity_xi_bear.csv` - from `ms_commodit_temp_2.ipynb`, for the
## Key constants

- `N_FACTORS = 6` (MST-comparable), `R_TOTAL_AH` (AH-selected, computed at
  runtime - was 2 on this data), `R_TOTAL = max(6, R_TOTAL_AH)`.
- `THETA_HAT = 0.125` - inherited from the characteristics universe, not
  re-estimated on this data.
- `K_STARTS = 20` multi-start EM initializations for the main fits; D10b/D10c
  robustness variants use only the first 10 starts (cheaper, since they're
  sensitivity checks, not the primary estimate).
