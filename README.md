# Renewable Energy and Electricity Demand Dataset for Generator Expansion Planning (GEP)

## Overview
This repository packages **hourly (8760-row) time series** that can be used to build **scenario-based Generator Expansion Planning (GEP)** models.  
The core workflow is:

1. **Raw data**: clean + synchronize timestamps + standardize units + derive renewable availability.
2. **Processed data**: Split the data into each group (Seasonal and Time).
3. **Gaussian Copula scenario sampling**: generate realistic multi-variable scenarios (load ↔ wind ↔ solar) while preserving dependence.
4. **Optimization results**: solve the GEP (e.g., MILP) and store build/dispatch/cost outcomes.

---

## Dataset Catalog

| Dataset | Location / Context | Category | Key Variables | Frequency | Time Span |
|---|---|---|---|---|---|
| **PJME Load** (`PJME_MW_1_year.csv`) | USA (PJM East) | Demand | Load (MW) | Hourly | 2018-01-01 → 2018-12-31 |
| **Wind SCADA** (`Wind_Speed_1_year.csv`) | Turkey (wind plant SCADA-style) | Renewable supply | Wind speed, direction, active power | Hourly | 2018-01-01 → 2018-12-31 |
| **Solar Irradiation (PVGIS)** (`Solar_Irradiation_1_year.csv`) | Ankara example / PVGIS-style series | Renewable resource | POA irradiance components, temp, wind | Hourly* | 2018-01-01 → 2018-12-31 |

\*Solar timestamps are hourly but recorded at **HH:10** (10 minutes past the hour) in PVGIS-style format; the processing step aligns them to the same hourly grid as the other datasets.

---

## Dataset Specifications (Data Dictionaries)

### 1) PJME Load (PJM East) — `PJME_MW_1_year.csv`
**Shape**: 8760 rows × 2 columns (no missing values)

| Column | Type | Description | Unit |
|---|---|---|---|
| `DATE_TIME` | datetime-like string | Timestamp (hourly) | — |
| `PJME_MW` | float | Electricity demand (system load) | MW |

---

### 2) Wind SCADA-style (Turkey) — `Wind_Speed_1_year.csv`
**Shape**: 8760 rows × 6 columns (no missing values)

| Column | Type | Description | Unit |
|---|---|---|---|
| `DateTime` | datetime-like string | Timestamp (hourly) | — |
| `LV ActivePower (kW)` | float | Measured active power output | kW |
| `Wind Speed (m/s)` | float | Hub-height (or measurement height) wind speed | m/s |
| `Theoretical_Power_Curve (KWh)` | float | Turbine power-curve output for the time step (often numerically equals kW for 1-hour averages) | kW (or kWh per hour) |
| `Wind Direction (°)` | float | Wind direction | degrees |
| `Energy_from_ActivePower (kWh)` | float | Energy computed from active power over the hour | kWh |

**Notes for GEP use**
- Convert power to **capacity factor** (CF) using `CF_wind = ActivePower / RatedPower`.
- Optionally use `Theoretical_Power_Curve` as a cleaned proxy when SCADA power is noisy.

---

### 3) Solar Irradiation (PVGIS-style) — `Solar_Irradiation_1_year.csv`
**Shape**: 8760 rows × 9 columns (no missing values)

| Column | Type | Description | Typical Unit (PVGIS convention) |
|---|---|---|---|
| `time` | string | Timestamp formatted like `YYYYMMDD:HHMM` (here minutes are always `10`) | — |
| `Gb(i)` | float | Beam (direct) irradiance on the plane-of-array (POA) | W/m² |
| `Gd(i)` | float | Diffuse irradiance on POA | W/m² |
| `Gr(i)` | float | Ground-reflected irradiance on POA | W/m² |
| `H_sun` | float | Solar elevation (sun height) | degrees |
| `T2m` | float | Air temperature at 2 m | °C |
| `WS10m` | float | Wind speed at 10 m | m/s |
| `Int` | int | PVGIS interval/flag field (constant in this extract) | — |
| `G(i)_POA` | float | Total POA irradiance (often ≈ `Gb(i)+Gd(i)+Gr(i)`) | W/m² |

**PVGIS Slope/Azimuth guidance (used to produce POA columns)**
- If you want the “best theoretical” POA irradiance for Ankara-like latitude (~39.6°N):  
  - **Azimuth** = `0°` (due south, PVGIS convention)  
  - **Slope/Tilt** ≈ `~32–35°` (rule-of-thumb: latitude − 5° to −10°)  
- Or simply select **Optimize slope and azimuth** in PVGIS when you do not need to match a physical roof/panel.

---

## GEP Workflow

### A) Raw Data (timestamp alignment + feature engineering)
**Goal**: produce a single aligned table suitable for scenario generation and optimization.

Typical steps:
- Parse timestamps and **synchronize to a common hourly index**.
  - Example: shift solar times from **HH:10 → HH:00** (or resample) so all sources align.
- Standardize units (kW → MW, irradiance to per-unit PV availability, etc.).
- Derive renewable availability:
  - `wind_cf(t)` from SCADA power or turbine curve.
  - `solar_cf(t)` from POA irradiance using a PV conversion model (simplified linear, PVWatts-style, or detailed).

**Recommended processed outputs**
- `processed/load_hourly.csv` (timestamp, load_MW)
- `processed/wind_hourly.csv` (timestamp, wind_speed, wind_power_MW, wind_cf)
- `processed/solar_hourly.csv` (timestamp, poa_Wm2, temp_C, solar_cf)

### B) Splitting Process (Seasonal + Day/Night Regimes)  
**Goal**: split the processed, timestamp-aligned dataset into **seasonal** and **day/night** subsets to enable:  
- regime-aware modeling (different patterns in summer vs winter, day vs night)  
- regime-conditioned scenario sampling (copula per regime)  
- targeted GEP stress tests (e.g., winter night = high load + low solar)  

**Splitting logic**  
- **Season split (calendar-based)**:  
  - **Spring**: Mar–May  
  - **Summer**: Jun–Aug  
  - **Fall**: Sep–Nov  
  - **Winter**: Dec–Feb  
- **Day/Night split (solar-driven)** *(recommended)*:  
  - **Day**: `H_sun > 0` **or** `G(i)_POA > 0`  
  - **Night**: `H_sun ≤ 0` **or** `G(i)_POA = 0`  
  - Use `H_sun` when available because it is physically interpretable and robust for PVGIS-derived series.  

Typical outputs:  
- `processing/Solar_Wind_Load_1_Year_spring_day.csv`  
- `processing/Solar_Wind_Load_1_Year_spring_night.csv`  
- `processing/Solar_Wind_Load_1_Year_summer_day.csv`  
- `processing/Solar_Wind_Load_1_Year_summer_night.csv`  
- `processing/Solar_Wind_Load_1_Year_fall_day.csv`  
- `processing/Solar_Wind_Load_1_Year_fall_night.csv`  
- `processing/Solar_Wind_Load_1_Year_winter_day.csv`  
- `processing/Solar_Wind_Load_1_Year_winter_night.csv`  
- `processing/Solar_Wind_Load_1_Year_WindScaled.csv` *(wind-scaled variant for sampling/training stability)*  

**Common split formats**  
- **Subset files**: each file contains only the rows that match its regime (season × day/night).  
- **Regime-tagged file (optional)**: one merged file with extra columns like `season` and `is_day` for filtering downstream.  

"""

### C) Gaussian Copula Scenario Sampling
**Goal**: generate multi-year or multi-scenario synthetic samples that preserve:
- each variable’s marginal distribution (load, wind_cf, solar_cf)
- cross-dependence (e.g., weather-driven correlation patterns)

Typical outputs:
- `scenarios/gaussian_copula/scenario_0001.csv`
- `scenarios/gaussian_copula/scenario_0002.csv`
- …
- `scenarios/gaussian_copula/summary.json` (marginals, correlation matrix, seed, scenario count)

**Common scenario formats**
- **Hourly scenarios**: each file contains 8760 rows.
- **Scenario cube**: one file with columns like `load_s1`, `wind_s1`, `solar_s1`, … (compact but wide).

---

### D) Optimization: Generator Expansion Planning (GEP)
**Goal**: choose new capacities (and optionally dispatch) that minimize total cost under uncertainty.

Typical optimization inputs:
- Scenario time series: `load(t,s)`, `wind_cf(t,s)`, `solar_cf(t,s)`
- Technology parameters: capex/opex, fuel costs, ramping, availability, emissions, storage efficiency
- Reliability constraints: reserve margin, loss-of-load probability approximation, or energy-not-served penalty

Typical outputs:
- `results/expansion_plan.csv` (new builds by technology)
- `results/dispatch_timeseries.csv` (generation by tech, by scenario)
- `results/cost_breakdown.csv` (capex, opex, fuel, penalties)
- `results/kpis.json` (renewable share, curtailment, emissions, EENS)

---

## Repository Layout (recommended)
```text
.
├── raw/
│   ├── PJME_MW_1_year.csv
│   ├── Wind_Speed_1_year.csv
│   └── Solar_Irradiation_1_year.csv
├── processed/
│   ├── Solar_Wind_Load_1_Year_spring_day.csv`
│   ├── Solar_Wind_Load_1_Year_spring_night.csv`
│   ├── Solar_Wind_Load_1_Year_summer_day.csv`
│   |── Solar_Wind_Load_1_Year_summer_night.csv`
|   |── Solar_Wind_Load_1_Year_fall_day.csv`
|   |── Solar_Wind_Load_1_Year_fall_night.csv`
|   |── Solar_Wind_Load_1_Year_winter_day.csv`
|   └── Solar_Wind_Load_1_Year_winter_night.csv`
|   
├── scenarios/
│   └── gaussian_copula/
│       ├── scenario_0001.csv
│       ├── scenario_0002.csv
│       └── summary.json
└── Optimization/
    ├── expansion_plan.csv
    ├── dispatch_timeseries.csv
    ├── cost_breakdown.csv
    └── kpis.json

