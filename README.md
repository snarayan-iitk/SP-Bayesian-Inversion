# How to Use This Code — Quick Guide

## 1. Prerequisites

- **Python 3.8+** with these packages installed:
  ```
  pip install numpy pandas matplotlib scipy emcee corner openpyxl
  ```
- **Windows/Excel input** supported natively (`.xlsx`). CSV also works.
- **Runtime expectation:** ~5–20 min per profile with `MCMC_WALKERS=500, MCMC_STEPS=3000`. Synthetic tests add another ~15 min.

## 2. Configure the Script (top of file)

- **`FILE_PATH`** — full path to your Excel/CSV file containing the SP profile(s).
- **`OUTPUT_DIR`** — folder where all PNG figures and diagnostics will be saved (auto-created).
- **`MCMC_WALKERS`** — number of ensemble walkers (default 500; keep ≥ 2× number of parameters).
- **`MCMC_STEPS`** — total MCMC iterations (default 3000). **If you see the emcee autocorrelation warning, raise this to 15000–20000 and `MCMC_BURN_IN` to ~6000.**
- **`MCMC_BURN_IN`** — samples discarded as burn-in (default 1000).
- **`MAX_SOURCES`** — maximum number of overlapping bodies to test per profile (default 1).
- **`GEOLOGY_TYPE`** — one of `"massive_sulfide"`, `"graphite"`, or `"unknown"`. This constrains the shape factor `q`.
- **`BOREHOLES_AVAILABLE`** — set `True` if you have borehole depth control, and set `BOREHOLE_DEPTH` accordingly. Adds a soft depth prior.

## 3. Prepare Your Input File

- One **row per station**; must contain:
  - an **X column** named one of: `Easting`, `UTMX`, `Station`, `X`, `Dist`, `Distance`
  - an **SP column** named one of: `SP`, `SP (mV)`, `SP (mV) Final`, `PD Corrected Final SP (mV)`
  - *(optional)* a **traverse / line column** (`Traverse`, `Traverse (Y)`, `Line`) → the script will auto-split into profiles.
- If no traverse column exists, the whole sheet is treated as one profile.
- Missing X/SP values are dropped; duplicate X values are removed automatically.

## 4. Run the Script

- From a terminal:
  ```
  python your_script_name.py
  ```
- You will be prompted:
  ```
  Do you want to run Synthetic Fidelity Tests (Cylinder, Sphere & Sheet) with Corner Plots? (y/n):
  ```
  - **`y`** → runs three synthetic validation tests first (ground truth = cylinder, sphere, sheet). **Do this at least once** to verify the inversion is behaving.
  - **`n`** → skips straight to processing your real data.

## 5. What the Script Does (per profile)

1. Loads the sheet and (optionally) splits by traverse.
2. Removes a linear regional trend.
3. Runs **Differential Evolution** to find a global starting point for each candidate number of sources.
4. Selects the best number of bodies using **BIC**.
5. Runs **MCMC (`emcee`)** for posterior sampling.
6. Produces and saves the following figures into `OUTPUT_DIR`:

| File | Contents |
|---|---|
| `<profile>_Inversion.png` | Data fit + R² + 2-panel subsurface cross-section |
| `<profile>_Convergence.png` | 9-panel diagnostics: R-hat, ESS, correlation matrix, trace & posterior plots for z, α, q |
| `<profile>_Corner_alpha_z_q.png` | Corner plot of α, z, q (single-body case only) |
| `Synthetic_*.png` (if `y`) | Validation fits and corner plots for the 3 test models |

## 6. What to Check in the Output

- **Convergence figure:**
  - R-hat bars must all be **< 1.1** (green).
  - ESS bars should exceed **100**.
  - Trace plots should look like "fuzzy caterpillars", not slow drifts.
- **Correlation matrix:** strong off-diagonal colours (|r| > 0.7) indicate K↔z↔q trade-offs — treat depth estimates with caution.
- **Console output:** the script prints a `DEPTH OVERESTIMATION DIAGNOSIS` block with correlations, CIs, and a warning if the profile is too short relative to depth.
- **R²** in the main figure — values > 0.9 indicate a good fit; low R² → try `MAX_SOURCES = 2` or increase MCMC steps.

## 7. Common Troubleshooting

- **`emcee.autocorr` warning ("chain shorter than 50×τ")** → increase `MCMC_STEPS` and `MCMC_BURN_IN`; consider fixing `q` if geology is known.
- **Depth hits the upper bound (500 m)** → profile may be too short; extend survey or reduce `MAX_SOURCES`.
- **R-hat > 1.1** → longer MCMC, or check that `GEOLOGY_TYPE` bounds are not too restrictive.
- **`File not found`** → check `FILE_PATH` uses a raw string: `r"C:\...\file.xlsx"`.
- **Figures not displaying** → if you're running headless (server/Jupyter without display), add `SHOW_PLOTS = False` (may need to be re-added manually) or set `matplotlib.use('Agg')` at the top.
- **Very slow runs** → lower `MCMC_WALKERS` to 200 and `MCMC_STEPS` to 2000 for a quick look; always re-run with full settings for final results.

## 8. Recommended Workflow

1. Run once with **synthetic tests enabled** to confirm installation and correctness.
2. Run on your real sheet with default settings.
3. Inspect `_Convergence.png` — if R-hat > 1.1 or ESS < 100, lengthen the chain and re-run.
4. Only trust the depth/angle outputs when diagnostics are green and the profile length is ≥ 3× the recovered depth.
