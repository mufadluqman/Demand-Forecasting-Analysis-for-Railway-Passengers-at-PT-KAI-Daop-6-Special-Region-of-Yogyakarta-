# Demand Forecasting Analysis for Railway Passengers at PT KAI Daop 6 (Special Region of Yogyakarta)

Six-month forecast (Sep 2026 – Feb 2027) of monthly railway passenger departures in DIY, using a seasonal ARIMA, a tuned Prophet, and their ensemble, evaluated under a locked protocol (rolling-origin CV plus a one-shot final test). Built in R. Data through Aug 2026.

![Forecast](output/figures/16_forecast_final.png)

## Key results

| | |
|---|---|
| **6-month forecast (ensemble)** | **≈ 6,327 thousand passengers** |
| vs. same period last year (5,955 thousand) | **+6.3%** |
| Peak month | Dec 2026, ≈ 1,168 thousand |
| Final-test MAPE (Mar–Aug 2026) | 5.9% – 7.3% across models (seasonal naive: 10.0%) |

The three models agree on the 6-month total (6,324 – 6,331 thousand) and differ mainly in the monthly pattern. Treat the numbers as indicative: see [Limitations](#limitations).

## Business problem

Capacity, scheduling, and resource planning for rail services need a demand outlook beyond the current month. This project answers: *how many passengers will depart from DIY each month from Sep 2026 to Feb 2027?*

## Data

- Monthly passenger departures, **Aug 2020 – Aug 2026 (73 months)**, in **thousand passengers**.
- Scope: departures in the DIY region from PT KAI, PT KCI, and PT Railink, according to the original source file. This is a combined series, not a single operator.
- The raw file was split across several tables with different layouts. They were merged into one series (overlapping years matched exactly; one missing value, Dec 2023, was filled from the overlapping table). Result: 73 consecutive months, no gaps or duplicates.
- Freight data was **not modeled** because it has an unexplained level break in Jan 2024 (≈ 51 to ≈ 18 thousand tons).

> **Data availability:** check that you are allowed to share the source data before committing it. The pipeline expects `data/raw/data_daop6.csv` with columns `bulan` (month-end date), `penumpang` (thousand passengers), `barang` (thousand tons).

## Approach

**Protocol (locked before modeling)**

- Training data: Aug 2020 – Feb 2026 (67 months). Final test: Mar – Aug 2026 (6 months), evaluated **once**, never used to choose or tune models.
- Model selection: rolling-origin CV inside the training data (36-month initial window, step 3, horizon 6; 9 windows, 54 forecasts per model). Metrics: MAE, RMSE, MAPE.
- After selection, refit on all 73 months and forecast 6 months ahead.

**Pipeline**

| Step | What | Script |
|---|---|---|
| 1 | Import, validate, save clean series | `01_impor.R` |
| 2 | EDA: trend, seasonality, STL, YoY | `02_eksplorasi.R` |
| 3 | SARIMA: ADF/KPSS → d, D; ACF/PACF → candidate orders; 14 models (6 candidates + `auto.arima`, each with/without holiday regressors); residual diagnostics; CV | `03_arima.R` |
| 4 | Prophet: 24-configuration grid (changepoint prior, seasonality mode, Fourier terms, holidays); CV | `04_prophet.R` |
| 5 | Compare all models on the same CV windows | `05_evaluasi.R` |
| 6 | One-shot final test, refit, final forecast | `06_forecast_final.R` |

**Holiday variables.** Because the data is monthly, holidays enter as two regressors counting days per month: `lebaran` (Eid al-Fitr ±7 days) and `nataru` (20 Dec – 4 Jan). Window lengths are design assumptions. Ramadan is not modeled.

**Selection rule.** Lowest CV RMSE among models whose residuals pass the Ljung-Box test (p > 0.05).

## Findings

**EDA**

- Demand grew roughly 11× (≈ 98 → ≈ 1,098 thousand, Aug 2020 → Aug 2026), highest value ≈ 1,182 thousand in Dec 2025.
- STL strength: trend 0.968, seasonality 0.545 (scale 0–1). Box-Cox λ ≈ 1.04, so no transformation is needed.
- Level series is non-stationary (ADF p = 0.979, KPSS p ≤ 0.01 on training data); one regular difference suffices (d = 1, D = 0). A positive ACF spike at lag 12 points to a seasonal AR term.
- Level shifts are suggested around May 2021, Apr 2022, Apr 2023, and Mar 2025 (indicative only; a mean-shift model slices a trending series into steps).

![Trend](output/figures/01_tren_penumpang.png)

**Selected models**

- **SARIMA(4,1,0)(1,0,0)[12]**, seasonal ARIMA with a seasonal AR term. Holiday regressors did not improve CV RMSE for any of the 7 specifications, so the final model uses none.
- **Prophet**: changepoint prior 0.5, additive seasonality, 4 Fourier terms, with holiday regressors.
- **Ensemble**: simple average of the two.

**Rolling CV (training data; MAE/RMSE in thousand passengers)**

| Model | MAE | RMSE | MAPE (%) |
|---|---|---|---|
| **Ensemble** | 59.1 | 75.3 | 6.64 |
| SARIMA | 68.0 | 81.5 | 7.49 |
| Prophet (tuned) | 60.4 | 82.6 | 6.85 |
| `auto.arima` (re-selected every window) | 88.6 | 113 | 9.79 |
| Prophet (default settings) | 165 | 188 | 18.2 |

Tuning matters: default Prophet is by far the weakest, and the tuned version is on par with SARIMA. SARIMA and tuned Prophet are tied on RMSE; the ensemble is best of the locked set.

**Final test, Mar – Aug 2026 (6 points)**

| Model | MAE | RMSE | MAPE (%) | Bias |
|---|---|---|---|---|
| Prophet (tuned) | 61.5 | 74.1 | 5.88 | +0.2 |
| Ensemble | 64.2 | 91.3 | 6.11 | -28.9 |
| SARIMA | 77.1 | 115 | 7.30 | -58.1 |
| Seasonal naive (reference) | 106 | 149 | 10.0 | -47.9 |

Every model beats seasonal naive. The largest error is Mar 2026 (actual 1,049; SARIMA 799; Prophet 916), coinciding with Eid al-Fitr on 21 Mar 2026. The ranking differs from CV, so there is no clear winner; the primary model was chosen from CV, not from the final test.

## Forecast (thousand passengers)

| Month | SARIMA | Prophet | **Ensemble** | Same month last year |
|---|---|---|---|---|
| Sep 2026 | 1,023 | 1,008 | **1,016** | 951 |
| Oct 2026 | 1,064 | 1,041 | **1,052** | 983 |
| Nov 2026 | 1,037 | 1,035 | **1,036** | 971 |
| Dec 2026 | 1,155 | 1,181 | **1,168** | 1,182 |
| Jan 2027 | 1,089 | 1,082 | **1,085** | 1,037 |
| Feb 2027 | 964 | 978 | **971** | 831 |
| **Total** | 6,331 | 6,324 | **6,327** | 5,955 |

Rounded values; 80% prediction intervals for SARIMA and Prophet are in `output/tables/forecast_final.csv`. The ensemble has no valid interval.

## Limitations

- **Short history, pandemic recovery.** 73 observations, starting in the recovery period; this affects trend and seasonality estimates and the earliest CV windows.
- **Holidays.** Window lengths are assumptions; Eid dates for 2021–2025 were not verified against official sources (2026 was); only six Eid events exist; Ramadan is not modeled. Feb 2027 (+17% vs Feb 2026) is the least reliable month.
- **Selection optimism.** 14 SARIMA models and 24 Prophet configurations were selected on the same CV that is reported, so CV scores lean optimistic. SARIMA candidate orders were read from ACF/PACF of the full series. `auto.arima` is the more honest benchmark.
- **Evaluation.** CV windows overlap (forecasts are not independent); no formal significance test between models; the final test has 6 points and is dominated by one event (Eid 2026). A Prophet configuration with the lowest CV RMSE failed residual checks and did not hold up in the final test, so it was kept as a reference only.
- **Intervals.** Prophet intervals are unreliable (autocorrelated, marginally non-normal residuals); its Ljung-Box p-value is approximate.
- **Possible optimism.** The forecast growth (+6.3%) exceeds the recent trend (≈ +4.7%, Mar–Aug 2026 vs 2025).
- **Statistical projection only.** No tariff changes, operational disruptions, or special events are modeled. The series records realized departures, not unconstrained demand.

## Repository structure

```
kai-daop6-forecasting/
├── data/
│   ├── raw/data_daop6.csv        # monthly series (see Data availability)
│   └── processed/df.rds          # cleaned series
├── R/
│   ├── 00_setup.R                # packages, folders, KAI palette, ggplot theme
│   ├── protokol.R                # locked protocol: split, CV settings
│   ├── kalender.R                # holiday regressors (Eid, Christmas/New Year)
│   ├── fungsi_ts.R               # helper functions (ACF/PACF plot, error metrics)
│   ├── 01_impor.R
│   ├── 02_eksplorasi.R
│   ├── 03_arima.R
│   ├── 04_prophet.R
│   ├── 05_evaluasi.R
│   ├── 06_forecast_final.R
│   ├── 06_uji_statistik.R        # exploratory statistical tests (first iteration)
│   └── 07_cv_sensitivitas.R      # exploratory CV sensitivity check (first iteration)
└── output/
    ├── figures/                  # all charts (no titles by design)
    └── tables/                   # CV results, final test, forecasts
```

## How to reproduce

1. Install R (developed with R 4.6 on Windows) and the packages: `tidyverse`, `lubridate`, `scales`, `forecast`, `prophet`, `tseries`, `strucchange`. (`prophet` needs a working Stan toolchain; installation can be slow.)
2. Open the `.Rproj` file and place the data at `data/raw/data_daop6.csv`.
3. Run in order: `01_impor.R`, `02_eksplorasi.R`, `03_arima.R`, `04_prophet.R`, `05_evaluasi.R`, `06_forecast_final.R` (`source("R/<file>")`). Steps 3 and 4 take several minutes.
4. Run `06_forecast_final.R` only once as designed: it is the one-shot final test.

Charts follow a "consulting minimalist" style: KAI palette, sans-serif fonts, white background, light horizontal grid only, no chart titles (captions are written in the report).

## Author

Mufad
