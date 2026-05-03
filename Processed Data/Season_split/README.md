# Season Split Dataset

This folder contains the processed annual wind, solar, load, and solar-capacity-adjusted time-series dataset split into four meteorological seasons:

- `winter`
- `spring`
- `summer`
- `fall`

The dataset is prepared for forecasting, data mining, AI modelling, mathematical optimization, and intelligent decision-support analysis.

---

## Folder Contents

| File | Description |
|---|---|
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_winter.csv` | Winter subset: December, January, February |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_spring.csv` | Spring subset: March, April, May |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_summer.csv` | Summer subset: June, July, August |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_fall.csv` | Fall subset: September, October, November |
| `season_numeric_season_stats.csv` | Min, mean, and max statistics for each numeric variable by season |
---

## Season Definition

The split uses Northern Hemisphere meteorological seasons.

| Season | Months |
|---|---|
| Winter | December, January, February |
| Spring | March, April, May |
| Summer | June, July, August |
| Fall | September, October, November |

---

## Data Columns

| Column | Description |
|---|---|
| `DateTime` | Hourly timestamp |
| `load_PJME_MW` | PJME load demand in MW |
| `solar_G_i__POA` | Plane-of-array solar irradiance |
| `solar_T2m` | Solar/weather temperature at 2 meters |
| `wind_Energy_scaled_MW` | Scaled wind power output in MW |
| `eta_eff` | Effective solar efficiency |
| `P_150MW_adj_MW` | Adjusted solar power output for the 150 MW case |
| `P_110MW_adj_MW` | Adjusted solar power output for the 110 MW case |

---

## Row Count Validation

The original dataset represents one complete non-leap hourly year.

| Dataset | Rows / Hours |
|---|---:|
| Winter | 2,160 |
| Spring | 2,208 |
| Summer | 2,208 |
| Fall | 2,184 |
| Total | 8,760 |

The total row count equals:

```text
2,160 + 2,208 + 2,208 + 2,184 = 8,760 hours
