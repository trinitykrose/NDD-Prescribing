# NDD-Prescribing
# Practice-level prescribing of neurodegenerative drugs in English general practice

Multilevel count models of monthly GP-practice prescribing, using NHS open data.
Practices are nested within ICBs, and models are fitted by disease, drug type
and individual drug.

## Pipeline

Run in order. Notebooks are Python, the modelling file is R.

| # | File | Purpose | Approx. run time |
|---|------|---------|------------------|
| 1 | `1_data_download.ipynb` | Downloads and processes the ODS practice list (EPRACCUR), BNF codes, IMD, ONSPD and GeoJSON boundaries | ~5 minutes |
| 2 | `2_EPD_download.ipynb` | Downloads English Prescribing Data (EPD), merges all years, filters to relevant items | ~3 hours |
| 3 | `3_GP_reg_wf_download.ipynb` | Downloads and processes GP registered patients and GP workforce | ~15 mins |
| 4 | `4_zero_grid.ipynb` | Builds the practice × month × drug grid, merges all sources, interpolates, applies floors, writes modelling datasets and exclusion-check `.tex` tables | ~15 mins |
| 5 | `5_modelling.rmd` | Fits models with `glmmTMB`/`brms`, runs diagnostics and sensitivity analyses, writes Excel results | ~15 hours |
| 6 | `6_tables_figures.ipynb` | Produces descriptive tables, GVIF table, choropleth maps and model/sensitivity tables | ~15 minutes |

## Data sources

- NHSBSA Open Data Portal: English Prescribing Data (EPD) and BNF codes
- NHS England ODS: `epraccur` (practice details)
- NHS Digital: Patients Registered at a GP Practice; General and Personal Medical Services (workforce)
- Index of Multiple Deprivation (IMD 2025), ONS Postcode Directory (ONSPD), ONS GeoJSON boundaries

All data is downloaded by the notebooks and is also included in this repo
(`Data/`) so results can be reproduced without re-downloading.

## Folder structure

```
Data/
  Raw/         source files as downloaded (incl. ONS, GeoJSON)
  Interim/     intermediate files from merging EPD years
  Processed/   cleaned single-source files (e.g. bnf_processed.csv, parquet files)
  Merged/      grid_presc_gp.parquet, df_full_floors.parquet
  Final/       df_full_modelling.parquet
    disease/       <name>_2level.parquet
    drug_type/     <name>_2level.parquet
    individual/    <name>_2level.parquet
Models/
  Primary_NB2/              primary models (<level>/ model_*.rds, results_*.rds, NB2_<level>_results.xlsx)
  Primary_NB1/              NB1 sensitivity
  Primary_Poisson/          Poisson sensitivity (Apomorphine only)
  AR1_NB2/                  AR(1) sensitivity
  Offset_Proportionality/  offset proportionality check
  Covariate_Sensitivity/    models without rurality / GP FTE
  IMD_Decile_Sensitivity/   IMD decile in place of scaled IMD
  Correlated_RE_brms/       correlated random intercepts (brms)
  RE_Normality/             random-effect normality checks
  RE_Covariate_Correlation/ random effects vs covariates
  RE_Stage_Correlation/     correlation between model stages
Tables/        LaTeX tables and gvif_by_dataset.csv
Figures/       Choropleth_maps
```

## Models

Outcome is monthly items per practice, with `log(expected cases)` as an offset.

- **Predictors:** scaled IMD, rurality, practice mean GP FTE, within-practice GP FTE, linear and quadratic time, working days, month
- **Random effects:** `(1 | ICB/Practice)`
- **Primary:** NB2. A hurdle (zero-truncated NB2 plus a zero part) is used where zeros are common, otherwise plain NB2.
- **Sensitivity:** NB1, Poisson, AR(1) practice-level errors, offset proportionality, dropping rurality or GP FTE, IMD decile, correlated random effects (`brms`)
- **Diagnostics:** VIF/GVIF, zero calibration, residual ACF, random-effect normality and correlations, variance partition coefficients (VPC)

## Outputs

- `Data/Final/<level>/<name>_2level.parquet`: modelling datasets
- `Models/*/`: fitted models (`.rds`) and Excel results (diagnostics, coefficients, VPCs)
- `Tables/`: descriptives, GVIF, primary model and sensitivity tables (`.tex`)
- `Figures/Choropleth_maps`: choropleth maps (`.png`, `.pdf`)

## Requirements

**Python:** pandas, numpy, pyarrow, requests, beautifulsoup4, statsmodels, scipy, geopandas, matplotlib, openpyxl

**R:** arrow, glmmTMB, brms, cmdstanr (with CmdStan installed), posterior, DHARMa, performance, lmtest, sandwich, dplyr, purrr, tibble, lubridate, timeDate, parallel, pbmcapply, writexl

## Setup

1. Set `BASE_DIR` at the top of each notebook and in `5_modelling.rmd` to your folder.
2. Run notebooks 1 to 4, then `5_modelling.rmd`, then notebook 6.

## Notes

- Download notebooks skip files that already exist. Delete the file first if you want to re-download it.
- Download links and the latest available months change over time, so a fresh download may differ slightly from the data included here.
- Model fits are cached as `.rds`. Set `FORCE_REFIT <- TRUE` to refit.
- Poisson was only run for apomorphine, to check convergence
- The `brms` models need CmdStan installed (`cmdstanr::install_cmdstan()`).
- Full runs are slow. Adjust `N_CORES`
