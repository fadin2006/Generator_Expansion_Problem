# Gaussian Correlation Matrices

This folder contains the **Gaussian-space correlation matrices** `Σ` used by the Gaussian copula sampler to generate correlated random draws of **solar irradiance**, **wind power availability**, and **system electric demand** for each season / time-of-day group.

The methodology follows:

> **H. Park and R. Baldick (2020)**, *Optimal capacity planning of generation system integrating uncertain solar and wind energy with seasonal variability*, Electric Power Systems Research 180, 106072. https://doi.org/10.1016/j.epsr.2019.106072

## 1. Background

The paper models the three stochastic inputs — solar irradiance ξ̃ₛ, wind availability ξ̃ᵥᵥ, and system load ξ̃_d — as **correlated random variables** with fixed marginal distributions (Johnson translation systems) and a fixed linear-correlation structure observed in the historical data.

To sample from such a joint distribution, a **Gaussian copula** is used:

1. Compute the empirical (data-space) correlations `ρ_data` between the three variables per season/time group.
2. Map each data-space correlation `ρ_data` to a Gaussian-space correlation `ρ_gauss` such that, after the inverse-CDF / CDF transforms of the copula, the generated samples reproduce the target data-space correlation. This follows Eqs. (2)–(3) of the paper, i.e.

   E[ξ̃ₛ ξ̃ᵥᵥ] = ∫∫ F̄ₛ⁻¹(Φ(zₛ)) · F̄ᵥᵥ⁻¹(Φ(zᵥᵥ)) · φ_{ρ}(zₛ, zᵥᵥ) dzₛ dzᵥᵥ

3. Assemble the 3×3 Gaussian correlation matrix Σ (Eq. 1 of the paper):

   ```
         ⎡   1       ρ_SW    ρ_SD ⎤
   Σ  =  ⎢ ρ_SW      1       ρ_WD ⎥
         ⎣ ρ_SD    ρ_WD      1    ⎦
   ```

4. Draw standard-normal vectors `z ~ N(0, Σ)`, map through `u = Φ(z)` to uniform marginals, then invert each variable's fitted Johnson CDF `F̄⁻¹(u)` to obtain correlated physical samples.

This folder stores **step 3** — the final Gaussian-space Σ for every season × time-of-day group and for each of the two candidate solar-PV pipelines (110 MW and 150 MW name-plate capacity).

> **Note on night groups.** As in the paper, solar PV generation during defined night hours (20:00–07:00) is fixed at zero, so the solar↔wind and solar↔demand entries for all `*_night` groups are set to 0 (NA in the original paper's Table 1).

## 2. Files

| File | Format | Description |
|---|---|---|
| `sigma_all_groups_long.csv` | Long / tidy | One row per matrix entry, easy to group and plot. |
| `sigma_all_groups_wide.csv` | Wide | One row per group, giving only the three off-diagonal correlations (ρ_SW, ρ_SD, ρ_WD). |

Both files contain the **eight groups** (4 seasons × {day, night}) for **both pipelines** (110 MW, 150 MW) — 16 group × pipeline combinations in total.

### 2.1 `sigma_all_groups_long.csv`

| Column | Type | Description |
|---|---|---|
| `pipeline` | str | Candidate solar-PV capacity the Σ was built for: `110MW` or `150MW`. |
| `group` | str | Season × time-of-day label: `{winter, spring, summer, fall}_{day, night}`. |
| `var_i` | str | Row variable of Σ: `Solar`, `Wind`, or `Demand`. |
| `var_j` | str | Column variable of Σ: `Solar`, `Wind`, or `Demand`. |
| `rho_gauss` | float | Gaussian-space correlation entry `Σ[var_i, var_j]` ∈ [-1, 1]. Diagonal entries are 1.0. |

Each group contributes 9 rows (full 3×3 matrix).

### 2.2 `sigma_all_groups_wide.csv`

| Column | Type | Description |
|---|---|---|
| `pipeline` | str | `110MW` or `150MW`. |
| `group` | str | Season × time-of-day label. |
| `rho_SW` | float | Solar–Wind Gaussian correlation. |
| `rho_SD` | float | Solar–Demand Gaussian correlation. |
| `rho_WD` | float | Wind–Demand Gaussian correlation. |

This is the compact form — one row per group × pipeline — convenient for quick inspection.

## 3. Quick-look values

The eight ρ_gauss triplets are (identical across both pipelines to within rounding, since Σ depends on the marginals of the *input* series, not on the PV candidate size; small differences appear only because each pipeline re-fits the marginals on slightly different data windows):

| Group | ρ_SW | ρ_SD | ρ_WD |
|---|---|---|---|
| winter_day   | −0.0226 |  0.1611 | −0.0380 |
| winter_night |   0.0    |  0.0    | −0.0031 |
| spring_day   | −0.0298 |  0.1899 |  0.0279 |
| spring_night |   0.0    |  0.0    |  0.0391 |
| summer_day   | −0.0032 |  0.0578 |  0.0437 |
| summer_night |   0.0    |  0.0    |  0.0520 |
| fall_day     | −0.0097 |  0.1715 |  0.0505 |
| fall_night   |   0.0    |  0.0    |  0.0673 |

## 4. How to reload Σ in Python

```python
import pandas as pd
import numpy as np

df = pd.read_csv("sigma_all_groups_long.csv")

# Rebuild Σ for one pipeline × group:
sub = df[(df["pipeline"] == "150MW") & (df["group"] == "fall_day")]
order = ["Solar", "Wind", "Demand"]
Sigma = (
    sub.pivot(index="var_i", columns="var_j", values="rho_gauss")
       .reindex(index=order, columns=order)
       .to_numpy()
)
# Sigma is now a 3x3 positive semi-definite matrix ready for np.random.multivariate_normal
```

## 5. Relation to the other folders in this repository

- **`../raw_110MW/`** — CSVs of the actual copula-generated samples for the 110 MW PV pipeline, drawn using the Σ matrices above.
- **`../raw_150MW/`** — same, for the 150 MW PV pipeline.
