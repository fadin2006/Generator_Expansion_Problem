# Renewable Energy and Electricity Demand Dataset for Generator Expansion Planning (GEP)

## Overview
This repository packages **hourly (8760-row) time series** that can be used to build **scenario-based Generator Expansion Planning (GEP)** models.  
The core workflow is:

1. **Raw data** (`raw/`): three original CSV source files — load, wind SCADA, and solar irradiation.
2. **Cleaned data** (`Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls`): timestamps synchronized, units standardized, and key features extracted into a single aligned table.
3. **Capacity-adjusted data** (`Data_Wind_Solar_Load_Capacity_Adjusted_MW.xls`): renewable outputs converted to MW using generator-rated capacities (Wind 200 MW, Solar 150 MW / 110 MW) and load scaled accordingly.
4. **Splitting**: seasonal × day/night regime subsets for copula-based scenario sampling.
5. **Gaussian Copula scenario sampling**: generate realistic multi-variable scenarios preserving dependence.
6. **Optimization**: solve the GEP (e.g., MILP) and store build/dispatch/cost outcomes.

---

## Dataset Catalog

### Raw Source Files

These are the three original CSV files uploaded to the repository (shown in the `raw/` folder):

| File | Location / Context | Category | Key Variables | Frequency | Time Span |
|---|---|---|---|---|---|
| **`PJME_MW_1_year.csv`** | USA (PJM East) | Demand | Load (MW) | Hourly | 2018-01-01 → 2018-12-31 |
| **`Wind_Speed_1_year.csv`** | Turkey (wind plant SCADA-style) | Renewable supply | Wind speed, direction, active power | Hourly | 2018-01-01 → 2018-12-31 |
| **`Solar_Irradiation_1_year.csv`** | Ankara / PVGIS-style series | Renewable resource | POA irradiance components, temp, wind | Hourly | 2018-01-01 → 2018-12-31 |

### Processed Files

| File | Description | Shape |
|---|---|---|
| **`Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls`** | Merged, timestamp-aligned, cleaned hourly data | 8 760 × 4 |
| **`Data_Wind_Solar_Load_Capacity_Adjusted_MW.xls`** | Generator-capacity-adjusted outputs in MW | 8 760 × 8 |

---

## Data Dictionaries

### 1) Raw: PJME Load — `PJME_MW_1_year.csv`
**Shape**: 8 760 rows × 2 columns

| Column | Type | Description | Unit |
|---|---|---|---|
| `DATE_TIME` | datetime string | Timestamp (hourly) | — |
| `PJME_MW` | float | Electricity demand (system load) | MW |

### 2) Raw: Wind SCADA — `Wind_Speed_1_year.csv`
**Shape**: 8 760 rows × 6 columns

| Column | Type | Description | Unit |
|---|---|---|---|
| `DateTime` | datetime string | Timestamp (hourly) | — |
| `LV ActivePower (kW)` | float | Measured active power output | kW |
| `Wind Speed (m/s)` | float | Hub-height wind speed | m/s |
| `Theoretical_Power_Curve (KWh)` | float | Turbine power-curve output | kW |
| `Wind Direction (°)` | float | Wind direction | degrees |
| `Energy_from_ActivePower (kWh)` | float | Energy from active power over one hour | kWh |

### 3) Raw: Solar Irradiation — `Solar_Irradiation_1_year.csv`
**Shape**: 8 760 rows × 9 columns

| Column | Type | Description | Unit |
|---|---|---|---|
| `time` | string | PVGIS timestamp `YYYYMMDD:HHMM` (minutes always `10`) | — |
| `Gb(i)` | float | Beam (direct) irradiance on POA | W/m² |
| `Gd(i)` | float | Diffuse irradiance on POA | W/m² |
| `Gr(i)` | float | Ground-reflected irradiance on POA | W/m² |
| `H_sun` | float | Solar elevation | degrees |
| `T2m` | float | Air temperature at 2 m | °C |
| `WS10m` | float | Wind speed at 10 m | m/s |
| `Int` | int | PVGIS interval flag | — |
| `G(i)_POA` | float | Total POA irradiance | W/m² |

### 4) Cleaned Data — `Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls`
**Shape**: 8 760 rows × 4 columns

| Column | Type | Description | Unit | Stats |
|---|---|---|---|---|
| `load_PJME_MW` | float | Hourly electricity demand | MW | mean 30 652, min 19 255, max 55 218 |
| `wind_Energy_from_ActivePower_kWh` | float | Wind turbine energy output per hour | kWh | mean 1 313, min 0, max 3 605 |
| `solar_G_i__POA` | float | Total plane-of-array irradiance | W/m² | mean 217, min 0, max 1 152 |
| `solar_T2m` | float | Ambient temperature at 2 m | °C | mean 12.5, min −13.9, max 33.8 |

### 5) Capacity-Adjusted Data — `Data_Wind_Solar_Load_Capacity_Adjusted_MW.xls`
**Shape**: 8 760 rows × 8 columns

This file converts the cleaned data into generator-level MW outputs using rated capacities and efficiency models.

**Generator Capacity Assumptions:**

| Technology | Rated Capacity | Conversion Method |
|---|---|---|
| **Wind** | 200 MW | Scale turbine output: `wind_scaled_MW = (ActivePower_kWh / max_ActivePower) × 200` |
| **Solar (150 MW)** | 150 MW | PV efficiency model: `P_150MW = η_eff × G_POA × 150 / G_STC` |
| **Solar (110 MW)** | 110 MW | PV efficiency model: `P_110MW = η_eff × G_POA × 110 / G_STC` |
| **Load** | — | Scaled proportionally to match system size |

| Column | Type | Description | Unit | Stats |
|---|---|---|---|---|
| `solar_T2m` | float | Ambient temperature | °C | mean 12.5, min −13.9, max 33.8 |
| `wind_scaled_MW` | float | Wind generation scaled to 200 MW rated capacity | MW | mean 66.7, min 0, max 183.0 |
| `load_scaled` | float | System load scaled to match generation portfolio | MW | mean 23 183, min 14 563, max 41 763 |
| `G_POA_Wm2` | float | Total POA irradiance | W/m² | mean 217, min 0, max 1 152 |
| `Ta_C` | float | Ambient temperature (same as solar_T2m) | °C | — |
| `eta_eff` | float | PV conversion efficiency (temperature-dependent) | — | mean 0.183, min 0.131, max 0.214 |
| `P_150MW_adj_MW` | float | Solar output for a 150 MW plant | MW | mean 47.6, min 0, max 139.4 |
| `P_110MW_adj_MW` | float | Solar output for a 110 MW plant | MW | mean 34.9, min 0, max 102.2 |

---

## GEP Workflow

### A) Raw Data → Cleaned Data

**Goal**: produce a single aligned table from the three raw CSV sources.

Steps performed:
1. Parse timestamps and synchronize to a common hourly index (solar times shifted from HH:10 → HH:00).
2. Merge load, wind energy, solar irradiance, and temperature into one table.
3. Handle missing values and validate 8 760 continuous hours.

**Input**: `PJME_MW_1_year.csv`, `Wind_Speed_1_year.csv`, `Solar_Irradiation_1_year.csv`  
**Output**: `Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls`

### B) Cleaned Data → Capacity-Adjusted Generator Data

**Goal**: convert raw measurements into MW-scale generator outputs suitable for GEP optimization.

**Wind (200 MW rated)**:
- Compute capacity factor from SCADA energy: `CF_wind(t) = Energy_kWh(t) / max(Energy_kWh)`
- Scale to rated capacity: `wind_scaled_MW(t) = CF_wind(t) × 200`

**Solar (150 MW and 110 MW rated)**:
- Compute temperature-dependent PV efficiency: `η_eff(t) = η_ref × [1 − β × (T_cell(t) − T_ref)]`
  - `η_ref ≈ 0.20` (reference efficiency at STC)
  - `β ≈ 0.004 /°C` (temperature coefficient)
  - `T_cell` estimated from ambient temperature and irradiance
- Compute power output: `P(t) = η_eff(t) × G_POA(t) × A_panel` scaled to rated capacity
- Two plant sizes provided: **150 MW** (`P_150MW_adj_MW`) and **110 MW** (`P_110MW_adj_MW`)

**Load**:
- Scaled proportionally to match the generation system size.

**Input**: `Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls`  
**Output**: `Data_Wind_Solar_Load_Capacity_Adjusted_MW.xls`

### C) Splitting Process (Seasonal + Day/Night Regimes)

**Goal**: split the capacity-adjusted dataset into seasonal and day/night subsets for regime-aware copula sampling.

**Season split** (calendar-based):
- Spring: Mar–May | Summer: Jun–Aug | Fall: Sep–Nov | Winter: Dec–Feb

**Day/Night split** (solar-driven):
- Day: `H_sun > 0` or `G_POA > 0`
- Night: `H_sun ≤ 0` or `G_POA = 0`

**Outputs** (8 subset files):
- `Solar_Wind_Load_1_Year_{season}_{day|night}.csv`

### D) Gaussian Copula Scenario Sampling

**Goal**: generate multi-scenario synthetic samples preserving marginal distributions and cross-variable dependence (load ↔ wind ↔ solar).

**Outputs**:
- `scenarios/gaussian_copula/scenario_NNNN.csv`
- `scenarios/gaussian_copula/summary.json`

### E) Optimization: Generator Expansion Planning (GEP)

**Goal**: choose new generation capacities that minimize total expected cost under uncertainty.

**Inputs**: scenario time series, technology parameters (capex/opex, fuel, emissions), reliability constraints.

**Outputs**:
- `results/expansion_plan.csv`
- `results/dispatch_timeseries.csv`
- `results/cost_breakdown.csv`
- `results/kpis.json`

---

## Repository Layout

```text
.
├── raw/
│   ├── PJME_MW_1_year.csv
│   ├── Wind_Speed_1_year.csv
│   └── Solar_Irradiation_1_year.csv
├── processed/
│   ├── Cleaned_Data_Wind_Solar_Load_With_Timestamp.xls
│   ├── Data_Wind_Solar_Load_Capacity_Adjusted_MW.xls
│   ├── Solar_Wind_Load_1_Year_spring_day.csv
│   ├── Solar_Wind_Load_1_Year_spring_night.csv
│   ├── Solar_Wind_Load_1_Year_summer_day.csv
│   ├── Solar_Wind_Load_1_Year_summer_night.csv
│   ├── Solar_Wind_Load_1_Year_fall_day.csv
│   ├── Solar_Wind_Load_1_Year_fall_night.csv
│   ├── Solar_Wind_Load_1_Year_winter_day.csv
│   └── Solar_Wind_Load_1_Year_winter_night.csv
├── scenarios/
│   └── gaussian_copula/
│       ├── scenario_0001.csv
│       ├── scenario_0002.csv
│       └── summary.json
└── results/
    ├── expansion_plan.csv
    ├── dispatch_timeseries.csv
    ├── cost_breakdown.csv
    └── kpis.json
```

---

## Citation
Please cite each dataset source individually using the provided BibTeX entries in the `references.bib` file.

---

For questions or collaborations, feel free to open an issue or contact the maintainer: fahrudin.muna@mail.ugm.ac.id
