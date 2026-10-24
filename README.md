# GiBleed causal inference: celecoxib vs diclofenac

RA Technical Assessment, Healthcare AI Initiative Research Center, Chapman University School of Pharmacy.

A new-user, active-comparator cohort study built from the synthetic OHDSI Eunomia GiBleed OMOP CDM (v5.5) tables. It estimates the 90-day risk of a recorded GI bleed for celecoxib (treatment = 1) versus diclofenac (treatment = 0), with crude and stabilized-IPTW-adjusted risk ratios and a confounding-distortion diagnostic, D = log(RR_crude) − log(RR_adjusted).

> The data are synthetic. Results are a methods exercise and must not be read as real-world comparative drug-safety evidence.

## Repository contents

| File | Purpose |
|---|---|
| `gibleed_analysis.ipynb` | Full analysis: cohort extraction, concept sets, attrition, baseline table, PS/IPTW, diagnostics, effect estimates, sensitivity analysis, DAG, interpretation, innovation proposal |
| `README.md` | This file |
| `.gitignore` | Keeps the data folder and local artifacts out of git |

The data folder `GiBleed_5.5/` is **not** in the repository (see below).

## Requirements

- Python 3.11
- Packages (versions used for the submitted results):

| Package | Version |
|---|---|
| pandas | 3.1.0 |
| numpy | 2.4.6 |
| statsmodels | 0.15.0 |
| scikit-learn | 1.9.1 |
| matplotlib | 3.11.2 |
| duckdb | 2.0.0 |
| ipykernel | 7.4.0 |

## How to run

1. **Get the data.** Unzip the supplied `GiBleed_5.5.zip` into the project root, so the CSVs sit at `GiBleed_5.5/PERSON.csv`, `GiBleed_5.5/DRUG_EXPOSURE.csv`, and so on.

2. **Create the environment.**

   ```bash
   conda create -n gibleed python=3.11
   conda activate gibleed
   pip install pandas==3.1.0 numpy==2.4.6 statsmodels==0.15.0 scikit-learn==1.9.1 matplotlib==3.11.2 duckdb==2.0.0 ipykernel==7.4.0
   ```

3. **Run the notebook.** Open `gibleed_analysis.ipynb` in VS Code or Jupyter, select the `gibleed` kernel, and run all cells top to bottom. No other input files are needed; every derived dataset is built inside the notebook from the raw CSVs.

## Approach

- **Cohort extraction:** DuckDB SQL queried directly over the OMOP CSVs.
- **Analysis:** pandas, statsmodels (logistic propensity-score model, risk-ratio estimation) and matplotlib (overlap and balance plots).

## AI use disclosure

Claude (Anthropic) was used as a tutor and coding assistant during this assessment: explaining methods, giving hints and reviewing code that I wrote, and helping with repository setup. All methodological decisions, code and written interpretation are my own, and I can explain each of them.
