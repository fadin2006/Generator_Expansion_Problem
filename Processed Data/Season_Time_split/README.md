# Season-Time Split Dataset

This folder contains the processed annual wind, solar, load, and solar-capacity-adjusted time-series dataset split into **8 season-time groups**:

- `winter_day`
- `winter_night`
- `fall_day`
- `fall_night`
- `summer_day`
- `summer_night`
- `spring_day`
- `spring_night`

The dataset is prepared for data mining, forecasting, AI modelling, mathematical optimization, and intelligent decision-support analysis.

---

## Folder Contents

| File | Description |
|---|---|
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_winter_day.csv` | Winter daytime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_winter_night.csv` | Winter nighttime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_fall_day.csv` | Fall daytime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_fall_night.csv` | Fall nighttime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_summer_day.csv` | Summer daytime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_summer_night.csv` | Summer nighttime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_spring_day.csv` | Spring daytime subset |
| `Data_Wind_Solar_Load_Capacity_Adjusted_MW_spring_night.csv` | Spring nighttime subset |
| `split_numeric_season_time_stats.csv` | Min, mean, and max statistics for each variable in each split |

---

## Split Definition

The dataset uses Northern Hemisphere meteorological seasons.

| Season | Months |
|---|---|
| Winter | December, January, February |
| Spring | March, April, May |
| Summer | June, July, August |
| Fall | September, October, November |

The day/night split is based on solar irradiance:

```text
day   = solar_G_i__POA > 1.0
night = solar_G_i__POA <= 1.0
