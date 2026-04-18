# Generation Expansion Optimization — Data Preparation

Renewable energy scenario data for stochastic generation expansion planning.  
**Location:** Ankara, Turkey (39.93°N, 32.86°E)  
**Period:** 1 year (8,760 hourly observations)  
**Framework:** Gaussian Copula → Scenario Reduction → Optimization

---

## Quick Start

```python
import pandas as pd

# Load a seasonal group
df = pd.read_csv("solar_Wind_Load_summer_day_1_year.csv", parse_dates=["DateTime"], index_col="DateTime")

# See what's inside
print(df.columns.tolist())
print(df.shape)
print(df.head())
```

---

## Repository Structure

```
├── Solar_Wind_Load_Scaled_Combined.csv      ← Full year (8,760 hours), all variables
├── solar_Wind_Load_{season}_{daynight}_1_year.csv  ← 8 seasonal group files
├── Seasonal_Energy_Profile_by_Group.csv     ← Efficiency & CF summary per group
├── Seasonal_Temperature_Statistics.csv      ← Temperature stats per group
├── renewable_energy_design_statistics.csv   ← Wind & solar design statistics
└── README.md
```

---

## File Descriptions

### 1. `Solar_Wind_Load_Scaled_Combined.csv`

The **master dataset** — all 8,760 hours in one file before seasonal splitting.

| Column | Unit | Description |
|--------|------|-------------|
| `DateTime` | — | Hourly timestamp (index) |
| `solar_G_i__POA` | W/m² | Plane-of-array solar irradiance from PVGIS |
| `solar_T2m` | °C | Ambient temperature at 2m height |
| `wind_Energy_scaled_MW` | MW | Wind farm output, scaled so max = 179 MW |
| `load_PJME_MW` | MW | Electrical load (PJM East × 0.08 scale) |
| `P_150MW_adj_MW` | MW | Adjusted solar output — 150 MW candidate plant |
| `P_110MW_adj_MW` | MW | Adjusted solar output — 110 MW existing plant |
| `eta_eff` | — | Hourly solar PV effective efficiency (0–1) |
| `eta_eff_pct` | % | Same as above, in percent |
| `CF_150MW` | — | Hourly capacity factor for 150 MW plant |
| `CF_110MW` | — | Hourly capacity factor for 110 MW plant |
| `season` | — | 0=Winter, 1=Fall, 2=Summer, 3=Spring |
| `day_night` | — | 0=Day (05:00–17:59), 1=Night |

---

### 2. Seasonal Group Files (8 files)

The master dataset split into **8 groups** by season × day/night:

| File | Season | Time | Typical rows |
|------|--------|------|-------------|
| `solar_Wind_Load_winter_day_1_year.csv` | Dec–Feb | 05:00–17:59 | ~1,170 |
| `solar_Wind_Load_winter_night_1_year.csv` | Dec–Feb | 18:00–04:59 | ~990 |
| `solar_Wind_Load_spring_day_1_year.csv` | Mar–May | 05:00–17:59 | ~1,196 |
| `solar_Wind_Load_spring_night_1_year.csv` | Mar–May | 18:00–04:59 | ~1,012 |
| `solar_Wind_Load_summer_day_1_year.csv` | Jun–Aug | 05:00–17:59 | ~1,196 |
| `solar_Wind_Load_summer_night_1_year.csv` | Jun–Aug | 18:00–04:59 | ~1,012 |
| `solar_Wind_Load_fall_day_1_year.csv` | Sep–Nov | 05:00–17:59 | ~1,183 |
| `solar_Wind_Load_fall_night_1_year.csv` | Sep–Nov | 18:00–04:59 | ~1,001 |

**All 8 files have the same columns** as the master dataset. These are the inputs for Part 2 (Gaussian Copula fitting).

**Why split?** Solar, wind, and demand behave very differently across seasons. Fitting one copula to the whole year would blur these differences. Separate fits per group capture the real seasonal patterns.

---

### 3. `Seasonal_Energy_Profile_by_Group.csv`

One row per group with efficiency and capacity factor summaries:

| Column | Description |
|--------|-------------|
| `Ta_avg_C` | Average ambient temperature (°C) |
| `G_avg_Wm2` | Average POA irradiance (W/m²) |
| `solar_eta_pct` | Solar PV effective efficiency (%) |
| `P_adj_avg_150MW` | Average adjusted solar output, 150 MW plant (MW) |
| `P_adj_max_150MW` | Maximum adjusted solar output, 150 MW plant (MW) |
| `CF_adj_150MW` | Capacity factor for 150 MW plant |
| `P_adj_avg_110MW` | Average adjusted solar output, 110 MW plant (MW) |
| `CF_adj_110MW` | Capacity factor for 110 MW plant |
| `wind_mean_MW` | Average wind output (MW) |
| `wind_max_MW` | Maximum wind output (MW) |
| `wind_CF` | Wind capacity factor |
| `load_mean_MW` | Average demand (MW) |

---

### 4. `Seasonal_Temperature_Statistics.csv`

Temperature summary per group:

| Column | Description |
|--------|-------------|
| `Avg_T2m` | Mean temperature (°C) |
| `Max_T2m` | Maximum temperature (°C) |
| `Min_T2m` | Minimum temperature (°C) |
| `Std_T2m` | Standard deviation (°C) |

---

### 5. `renewable_energy_design_statistics.csv`

Design-level statistics for the three renewable output columns:

| Column | Description |
|--------|-------------|
| `count` | Number of hours (8,760) |
| `min` | Minimum value (MW) |
| `min_nonzero` | Smallest positive value (MW) |
| `max` | Maximum value (MW) |
| `mean` | Average output (MW) |
| `std` | Standard deviation (MW) |
| `median` | Median output (MW) |
| `sum_MWh` | Total annual energy (MWh) |

---

## Plant Design Parameters

### Wind Farm (200 MW nameplate)

| Parameter | Value |
|-----------|-------|
| Design | 48 × Vestas V150-4.2 MW |
| Nameplate capacity | 200 MW |
| Loss factor | 10.5% |
| Effective maximum | 179 MW |
| Scaling method | Max-based (max output → 179 MW) |

### Solar PV (two designs)

| Parameter | 150 MW (candidate) | 110 MW (existing) |
|-----------|--------------------|--------------------|
| Nameplate | 150 MW | 110 MW |
| Module efficiency (η₀) | 22% | 22% |
| Inverter efficiency | 85% | 85% |
| PV area | 1,393,500 m² | 1,021,900 m² |
| Area rule | 9.29 m²/kW | 9.29 m²/kW |

### Solar Efficiency Model

From the paper (Eq. 4–5):

```
η_eff = η₀ × [1 − 0.0042 × (G/18 + Tₐ − 20)] × η_inv
P_solar = A × η_eff × G × 10⁻⁶   [MW]
```

Where:
- `G` = POA irradiance (W/m²)
- `Tₐ` = ambient temperature (°C)
- `A` = PV area (m²)

---

## How the Data Was Prepared

### Step-by-step pipeline (Part 1)

```
Raw CSVs (Wind, Solar, Load)
  │
  ├─ Step 1–4:  Load, parse timestamps, merge into one DataFrame
  ├─ Step 5:    Clean negatives and NaNs
  ├─ Step 6:    Scale wind → max = 179 MW (loss-adjusted)
  ├─ Step 7:    Solar PV model → P_150MW_adj_MW, P_110MW_adj_MW
  ├─ Step 8:    Scale load → × 0.08
  ├─ Step 9:    Validate: no value exceeds nameplate
  ├─ Step 10:   Calculate general efficiency + statistics
  ├─ Step 11:   Add season (0–3) and day_night (0/1) labels
  ├─ Step 12:   Split into 8 seasonal groups
  ├─ Step 13:   Per-group efficiency calculations
  └─ Step 14:   Save all files
```

### What happens next (Part 2)

```
8 seasonal group CSVs
  │
  ├─ Step 15–16: Wind zero smoothing for copula fitting
  ├─ Step 17–18: Fit marginal distributions (Johnson, Gamma, Weibull, etc.)
  ├─ Step 19:    Estimate Gaussian copula correlation Σ per group
  ├─ Step 20:    Generate 25 correlated scenarios per group
  ├─ Step 21:    Validate (nameplate + correlation preservation)
  └─ Step 24:    Reduce to 3 representative scenarios per group
```

---

## Key Constraints

These constraints are enforced throughout the pipeline:

| Constraint | Rule |
|------------|------|
| Wind cannot exceed 179 MW | Loss-adjusted nameplate (10.5% losses) |
| Solar 150 MW cannot exceed its seasonal max | Per-group clipping from data |
| Solar 110 MW cannot exceed its seasonal max | Per-group clipping from data |
| Nighttime solar = 0 | When POA irradiance ≤ 1.0 W/m² |
| No negative values | All power columns clipped to ≥ 0 |

---

## How to Use These Files

### For optimization / unit commitment

```python
# Load one seasonal group
df = pd.read_csv("solar_Wind_Load_summer_day_1_year.csv",
                 parse_dates=["DateTime"], index_col="DateTime")

# Extract the three variables for your model
solar_150 = df["P_150MW_adj_MW"]   # MW
wind      = df["wind_Energy_scaled_MW"]  # MW
demand    = df["load_PJME_MW"]     # MW
```

### For Gaussian copula (Part 2)

```python
# These files are the direct input to Part 2
# Part 2 fits marginals + copula + generates scenarios
# See: DataPreparation_Part2_Copula.ipynb
```

### For custom analysis

```python
# Load the full year
df_full = pd.read_csv("Solar_Wind_Load_Scaled_Combined.csv",
                      parse_dates=["DateTime"], index_col="DateTime")

# Filter by season
summer = df_full[df_full["season"] == 2]

# Plot duration curve
solar_sorted = df_full["P_150MW_adj_MW"].sort_values(ascending=False).values
plt.plot(range(len(solar_sorted)), solar_sorted)
plt.xlabel("Hours"); plt.ylabel("Solar output (MW)")
```

---

## Data Sources

| Variable | Source | Resolution |
|----------|--------|-----------|
| Solar irradiance (POA) | PVGIS 5.2 (SARAH database) | Hourly |
| Ambient temperature | PVGIS 5.2 | Hourly |
| Wind energy | SCADA turbine dataset | Hourly |
| Electrical load | PJM Interconnection (PJME) | Hourly |

---

## References

1. Sklar, A. (1959). *Fonctions de répartition à n dimensions et leurs marges.*
2. Dupačová, J. et al. (2003). *Scenario reduction in stochastic programming.* Mathematical Programming.
3. Papaefthymiou, G. & Kurowicka, D. (2009). *Using copulas for modeling stochastic dependence in power system uncertainty analysis.* IEEE Trans. Power Systems.
4. Heitsch, H. & Römisch, W. (2003). *Scenario reduction algorithms in stochastic programming.*

---

## License

This dataset is for academic research purposes.
