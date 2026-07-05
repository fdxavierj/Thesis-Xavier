# `commodity_equity_portfolios_by_characteristics.ipynb`

Constructs Block 2 of the test asset universe (Table 1, N=115): commodity-linked
equity portfolios from CRSP/Compustat via WRDS, sorted seven overlapping ways
on a pooled universe of direct producers + processors. This is a **data
construction** notebook, not an analysis notebook - its sole output feeds
`Input/test_asset_universe.csv` used by the MST and MS pricing notebooks.

## Requires WRDS access

This notebook makes live queries to WRDS (CRSP, Compustat, CCM link table) and
cannot be re-run without valid WRDS credentials. `WRDS_USER` is set in the
Section 0 config cell - **make sure to change to own WRDS username**.

Also requires:
- CFTC COT data (auto-downloaded; manual fallback URLs are in the Section 5A cell if the API call fails)
- `commodity_futures_data.xlsx` - LSEG Datastream near/next futures prices, used for the Basis-proxy (Section 6B)
- Fama-French daily risk-free rate (auto-downloaded in Section 11)

## Pipeline (Sections 1–14)

| Section | Does |
|---|---|
| 1 | Defines the commodity SIC-code universe (producers + processors) |
| 2 | Pulls CRSP monthly/daily, Compustat annual fundamentals, CCM link table |
| 3 | Computes book equity (Davis, Fama, French 2000 hierarchy) |
| 4 | Timing conventions - matches BE to ME |
| 5 | Loads and splices CFTC COT data (Legacy pre-Jun 2009 + Disaggregated post) for the HP-proxy |
| 6 | Assigns firm-level HP-proxy (from COT) and Basis-proxy (from Datastream futures curves) monthly |
| 7 | Builds the monthly sort panel |
| 8 | Runs all 7 overlapping sort schemes on the pooled universe |
| 9 | Computes daily value-weighted portfolio returns |
| 10 | Aggregates to weekly (Wednesday close) |
| 11 | Subtracts risk-free rate → weekly excess returns |
| 12 | Diagnostics (NaN coverage by scheme) |
| 13 | Saves `Output/commodity_equity_weekly_by_characteristics.csv` and portfolio counts |
| 14 | Builds the four long-short Pass 3 factors (BM, MOM, HP, BASIS) from the single-sort columns, saves `Output/commodity_factors_weekly.csv` |

## Sort schemes (115 portfolios total)

| Sort | Grid | N |
|---|---|---|
| BM × HP-proxy | 5×3 | 15 |
| MOM × BM | 5×5 | 25 |
| HP-proxy × MOM | 3×5 | 15 |
| BM × Basis-proxy | 5×3 | 15 |
| MOM × Basis-proxy | 5×3 | 15 |
| BM deciles | 1×10 | 10 |
| MOM deciles | 1×10 | 10 |
| HP-proxy quintiles | 1×5 | 5 |
| Basis-proxy quintiles | 1×5 | 5 |

A sixth double sort (HP-proxy × Basis-proxy, 3×3) is defined in the code but
commented out, as it is removed due to low coverage.


# `commodity_currency_portfolios.ipynb`

Constructs Block 4 of the test asset universe (Table 1, N=3): 12 commodity
currencies sorted by 24-month rolling beta to an equal-weighted commodity
index into tercile portfolios, plus the long-short factor rCURR.

## Section map

| Cell | Content |
|---|---|
| 1 | Configuration: 12 currency codes, inversion list (FCY-per-USD → USD-per-FCY), beta window (104 weeks) |
| 2 | Load currency data from `commodity_currencies_data.xlsx` |
| 3 | Weekly FX returns (Wednesday close, log-compounded) |
| 4 | Build the equal-weighted commodity index from Block 1's individual contract returns |
| 5 | 24-month rolling commodity beta per currency (manual `np.linalg.lstsq`, not `statsmodels` - no `RollingOLS` dependency) |
| 6 | Monthly tercile sort on beta, applied to next month's weekly returns |
| 7 | Subtract Fama-French weekly RF (aligned from Friday-close to Wednesday-close via daily forward-fill) |
| 8 | Save |

## Required inputs

- `commodity_currencies_data.xlsx` - Datastream FX export (same header-row
  parsing pattern as the futures notebook).
- `Output/commodity_returns_weekly.csv` - from
  `commodity_futures_portfolios.ipynb` (see fix #1 above - this dependency
  now resolves correctly).
- Fama-French weekly factors (auto-downloaded via `pandas_datareader`).

## Outputs (`Output/`)

- `commodity_currency_weekly.csv` - CUR1/CUR2/CUR3 tercile portfolios plus
  rCURR (CUR3 − CUR1), N=4 columns, weekly excess returns.

## Key constants

- `BETA_WINDOW = 104` weeks (~24 months), `MIN_OBS = 24` minimum valid
  observations per rolling regression.
- `STUDY_START = '1995-01-04'`, `STUDY_END = '2023-12-27'`.
- 10 of 12 currencies require inversion (quoted FCY-per-USD in the source
  data); AUD and NZD are the exceptions.



# `commodity_futures_portfolios.ipynb`

Constructs Block 1 of the test asset universe (Table 1, N=12): 27 commodity
futures sorted three ways (basis, momentum, hedging pressure) into
equal-weighted tercile portfolios, plus a fourth characteristic (value) and
the four long-short Pass 3 factors.

## Section map

| Section | Content |
|---|---|
| 1 | Configuration (27-contract `CONTRACT_MAP`, sample dates) |
| 2 | Load F1/F2 daily prices from `commodity_futures_data.xlsx`; fix known data errors (Heating Oil unit error, isolated-spike detector); Sugar No.11 spot-check |
| 3 | Weekly futures excess returns including daily-accrued roll yield; subtract Fama-French daily RF |
| 4 | Sort characteristics: Basis, Momentum, Value |
| 5 | Hedging Pressure (CFTC COT two-source splice - see `commodity_equity_portfolios_by_characteristics.ipynb`, this is a direct port of that notebook's COT logic) |
| 6 | Monthly tercile ranks |
| 7 | Equal-weighted tercile portfolios |
| 8 | Long-short Pass 3 factors (rBasis, rMOM, rHP, rValue) + diagnostics (sort directions, return sanity, sector plot) |
| 9 | Save outputs |

## Required inputs

- `commodity_futures_data.xlsx` - manual Datastream export, PS-series columns
  (row 4 header format, same parsing pattern as
  `commodity_equity_portfolios_by_characteristics.ipynb`).
- CFTC COT data (same source/splice logic as the equity notebook's Section 5).
- Fama-French daily factors (auto-downloaded from the data library).

## Outputs (`Output/`)

- `commodity_returns_weekly.csv` - individual contract-level weekly excess
  returns (27 contracts) - **consumed by `commodity_currency_portfolios.ipynb`**
  to build the commodity index used for currency beta sorts.
- `commodity_futures_weekly.csv` - the 12 tercile portfolios (Block 1).
- The four Pass 3 factor columns, folded into `commodity_futures_weekly.csv`
  (prefixed `r`) and separately combined into `pass3_factors.csv` by
  `commodity_asset_universe.ipynb`.

## Key constants

- `SAMPLE_START = '1990-01-01'`, `HP_START = '1995-01-01'` (CFTC COT
  availability), `END_DATE = '2023-12-31'`.
- 27 contracts across 5 sectors (Energy, Metals, Grains, Softs, Livestock).


# `commodity_asset_universe.ipynb`

Stacks Blocks 1 (futures), 2 (equities), 3 (currencies) into the
final N=130 test asset universe used by every MST/MS notebook downstream, and
separately saves the 4 Pass 3 factors restricted to the same date range.

## What this notebook does

1. Loads three CSVs from `Output/` (each produced by its own upstream
   notebook - see the corrected header comment for the mapping).
2. Drops the long-short factor columns from Blocks 1 and 4 (`rBasis`, `rMOM`,
   `rHP`, `rValue` from futures; `rCURR` from currencies) - only the tercile
   portfolios go into the pricing universe, the long-short factors are saved
   separately as Pass 3 observables instead.
3. Prefixes columns by block (`B1_`, `B2_`, `B4_`) and outer-joins on the
   Wednesday date index.
4. Restricts to `1996-02-07`–`2023-12-31` (currency data availability is the
   binding constraint - it starts latest of the three blocks).
5. Prints coverage diagnostics per block, saves the combined panel.
6. Separately loads and date-restricts `commodity_factors_weekly.csv` (built
   in `commodity_equity_portfolios_by_characteristics.ipynb`, Section 14) to
   produce `pass3_factors.csv`.

## Required inputs (all in `Output/`, i.e. run these three notebooks first)

- `commodity_futures_weekly.csv` ← `commodity_futures_portfolios.ipynb`
- `commodity_equity_weekly_by_characteristics.csv` ← `commodity_equity_portfolios_by_characteristics.ipynb`
- `commodity_currency_weekly.csv` ← `commodity_currency_portfolios.ipynb`
- `commodity_factors_weekly.csv` ← `commodity_equity_portfolios_by_characteristics.ipynb` (Section 14)

## Outputs (`Output/`)

- `test_asset_universe.csv` - the N=130 panel every MST/MS notebook reads as
  its primary test asset universe.
- `pass3_factors.csv` - the 4 observable Pass 3 factors (BM, MOM, HP, BASIS),
  date-restricted to match.
