# Copula Samples — 150 MW Candidate Solar PV Pipeline

This folder stores the **i.i.d. copula-generated samples** of three correlated random variables — **solar irradiance / PV availability**, **wind power availability**, and **system-wide electric demand** — used as node realizations at each stage of the multi-stage stochastic scenario tree.


The `150MW` label refers to the **candidate solar-PV farm name-plate capacity** used in this variant of the generator-expansion test system. This matches the **baseline value in the paper**: in Section 3.1 the authors size each candidate PV farm at 150 MW, which (with a PV module efficiency of 15 % and DC/AC conversion efficiency of 85 %) requires A = 1.3935 × 10⁶ m² of panel area, and Table 3 lists 3 600 MW of total candidate PV capacity = 24 × 150 MW PV farms. The 110 MW sensitivity study lives in `../raw_110MW/`.

## 1. Pipeline overview

For every `(season, time-of-day)` group the sampling pipeline is:

1. Load the historical hourly series for solar DNI / irradiance, wind availability and system load, partitioned by season and by day (08:00–19:00) / night (20:00–07:00) windows, following Section 2.1 of the paper.
2. Fit a parametric **Johnson translation-system** marginal to each of the three variables on that group's data.
3. Load the **Gaussian-space correlation matrix** `Σ` for that group (see `../Gaussian_Corellation/`), calibrated so that samples pushed through the copula reproduce the empirical data-space correlations (paper Eqs. (1)–(3), Tables 1–2).
4. Draw `N` standard-normal vectors `z ~ 𝒩(0, Σ)`; map through `u = Φ(z)`; invert through each fitted Johnson CDF `F̄⁻¹(u)` to obtain physical samples of (solar, wind, demand).
5. **Clip** each sample to the physical capacity cap so no draw exceeds installed / realized maximum:
   - solar samples clipped to the realized 150 MW PV-farm max (≈ name-plate × module & inverter efficiency × irradiance, per Eqs. (4)–(5))
   - wind clipped to the wind-farm name-plate
   - demand clipped to the peak scaled-system demand.
6. For all `*_night` groups, solar PV output is fixed to zero per Section 2.1 of the paper ("solar PV generation during the defined night hours is equal to zero").

Both the **unclipped** and **clipped** values are stored, so the effect of capacity capping on each draw is visible.

## 2. Files

One CSV per season × time-of-day group — **8 files**:

| File | Group (season · time) | Solar present? |
|---|---|---|
| `copula_spring_day_150MW.csv`   | Spring · Day   | Yes |
| `copula_spring_night_150MW.csv` | Spring · Night | No (fixed 0) |
| `copula_summer_day_150MW.csv`   | Summer · Day   | Yes |
| `copula_summer_night_150MW.csv` | Summer · Night | No (fixed 0) |
| `copula_fall_day_150MW.csv`     | Fall · Day     | Yes |
| `copula_fall_night_150MW.csv`   | Fall · Night   | No (fixed 0) |
| `copula_winter_day_150MW.csv`   | Winter · Day   | Yes |
| `copula_winter_night_150MW.csv` | Winter · Night | No (fixed 0) |

## 3. Column schema

All eight CSVs share the same schema:

| Column | Unit | Type | Description |
|---|---|---|---|
| `solar_sim` | MW | float | **Un-clipped** copula draw for solar-PV available power. For `*_night` groups this is 0 by construction. |
| `solar_clipped` | MW | float | `min(solar_sim, solar_cap_MW)` — the sample used downstream. |
| `wind_sim` | MW | float | Un-clipped copula draw for available wind power. |
| `wind_clipped` | MW | float | `min(wind_sim, wind_cap_MW)`. |
| `demand_sim` | MW | float | Un-clipped copula draw for system-wide electric demand. |
| `demand_clipped` | MW | float | `min(demand_sim, demand_cap_MW)`. |
| `solar_cap_MW` | MW | float | Clip threshold used for solar (0 for night groups). |
| `wind_cap_MW` | MW | float | Clip threshold used for wind (wind-farm name-plate capacity). |
| `demand_cap_MW` | MW | float | Clip threshold used for demand (system peak). |
| `group` | — | str | Season-time label, e.g. `fall_day`, `winter_night`. |
| `pipeline` | — | str | Always `150MW` in this folder. |

Each CSV contains **N sample rows** (one row = one joint draw of solar/wind/demand for that group).

## 4. Mapping to the paper

| Paper symbol | Column here |
|---|---|
| `P_sᵐᵃˣ(ξ^ω)` — realized available solar PV capacity (Eq. 19) | `solar_clipped` |
| `P_wᵐᵃˣ(ξ^ω)` — realized available wind capacity (Eq. 18)    | `wind_clipped` |
| `P_d(ξ^ω)`   — realized electric demand (Eq. 9)              | `demand_clipped` |
| `ξ̃_s, ξ̃_w, ξ̃_d` — random variables                         | `solar_sim`, `wind_sim`, `demand_sim` |

These clipped samples populate the stage-wise node realizations in the multi-stage scenario tree (Section 2.3 and Fig. 1 of the paper) before scenario reduction via `SCENRED2`.

## 5. Quick-look example

First rows of `copula_fall_day_150MW.csv`:

```
solar_sim, solar_clipped, wind_sim, wind_clipped, demand_sim, demand_clipped, solar_cap_MW, wind_cap_MW, demand_cap_MW, group,    pipeline
145.651,   145.651,       361.158,  178.927,      2815.306,   2815.306,       145.664,      178.927,     3681.84,       fall_day, 150MW
122.452,   122.452,       195.969,  178.927,      2616.109,   2616.109,       145.664,      178.927,     3681.84,       fall_day, 150MW
 33.434,    33.434,        66.309,   66.309,      2492.745,   2492.745,       145.664,      178.927,     3681.84,       fall_day, 150MW
```

Here `solar_cap_MW = 145.664 MW` is the *realized* (post-efficiency and post-ambient-temperature-derating) cap for a fall-day draw given the 150 MW name-plate farm, which is why `solar_clipped` can exceed 150 MW only at the AC rating — and why several draws (e.g. `wind_sim = 361.158 → wind_clipped = 178.927`) show visible capping on the wind column.

## 6. Reload in Python

```python
import pandas as pd

df = pd.read_csv("copula_fall_day_150MW.csv")
# The three realizations ready to plug into the stochastic program:
solar   = df["solar_clipped"].to_numpy()
wind    = df["wind_clipped"].to_numpy()
demand  = df["demand_clipped"].to_numpy()
```

## 7. Related folders

- **`../Gaussian_Corellation/`** — the Σ matrices used to generate these draws.
- **`../raw_110MW/`** — same pipeline, but with a smaller 110 MW candidate PV capacity (sensitivity study).
