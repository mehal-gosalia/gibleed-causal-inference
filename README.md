# GiBleed causal inference: celecoxib vs diclofenac

RA Technical Assessment, Healthcare AI Initiative Research Center, Chapman University School of Pharmacy.

A new-user, active-comparator cohort study built from the synthetic OHDSI Eunomia GiBleed OMOP CDM (v5.5) tables. It estimates the 90-day risk of a recorded GI bleed for celecoxib (treatment = 1) versus diclofenac (treatment = 0), with crude and stabilized-IPTW-adjusted risk ratios, sensitivity analyses and a confounding-distortion diagnostic, D = log(RR_crude) − log(RR_adjusted).

> The data are synthetic. Results are a methods exercise and must not be read as real-world comparative drug-safety evidence.

## Repository contents

| File | Purpose |
|---|---|
| `gibleed_analysis.ipynb` | The full, reproducible analysis, with code, outputs and written sections for Tasks 1–5 |
| `README.md` | Requirements, run instructions, results summary and methods audit |
| `.gitignore` | Keeps the data folder and local files out of git |

The data folder `GiBleed_5.5/` is **not** in the repository (see "How to run").

## Requirements

Python 3.11 with:

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

3. **Run the notebook.** Open `gibleed_analysis.ipynb`, select the `gibleed` kernel, then restart and run all cells top to bottom (about 1 minute; the bootstrap takes about 15 seconds). Every derived dataset is built inside the notebook from the raw CSVs.

## Where to find each deliverable

| Brief requirement | Notebook section |
|---|---|
| Table exploration, feasibility checks | Task 1 → Step 1 |
| Concept-set table (ID, name, role, inclusion rule) | Task 1 → Step 2 |
| Cohort flow / attrition table | Task 1 → Step 3 |
| Analysis dataset (one row per patient), data dictionary, crossover check | Task 1 → Step 4 |
| Target trial, estimand, assumptions, DAG, covariate justification | Task 2 |
| Treatment-stratified baseline table with SMDs | Task 3 → Step 5 |
| Propensity score model and overlap plot | Task 3 → Step 6 |
| Stabilized weights, extreme weights, ESS, Love plot | Task 3 → Step 7 |
| Crude and IPTW-adjusted RR and RD with 95% CIs | Task 3 → Step 8 |
| D, bootstrap CI for D, sensitivity analyses, E-value | Task 4 → Step 9 |
| Innovation proposal (370 words) | Task 4 → Methodological innovation |
| Scientific interpretation (397 words) | Task 5 |
| Methods audit, method selection, verification, two misleading results | Methods audit section |

## Results at a glance

**Cohort.** 2,694 initiators → **2,605** after exclusions (celecoxib 1,781; diclofenac 824). Only the 90-day observation requirement excluded anyone (89 patients); no patient had both drugs, a baseline GI bleed or prior study-drug use. Each concept set contained only its starting concept (celecoxib 1118084, diclofenac 1124300, GI hemorrhage 192671), and formulation codes lacked CONCEPT_ANCESTOR links to their ingredients (reported; no effect on this cohort).

| Analysis | Risk celecoxib | Risk diclofenac | RR (95% CI) | RD (95% CI) |
|---|---|---|---|---|
| Crude | 19.3% | 14.7% | 1.31 (1.08–1.59) | 4.6% (1.5–7.6%) |
| IPTW (stabilized) | 18.8% | 14.7% | 1.28 (1.05–1.55) | 4.1% (1.1–7.2%) |

- **Balance and positivity:** only baseline condition count was imbalanced before weighting (SMD 0.23); all weighted SMDs < 0.01. PS overlap was good (AUC 0.57), maximum weight 2.82, effective sample size 96–99%.
- **Confounding distortion:** D = 0.024 (bootstrap 95% CI −0.013 to 0.067), a small change that is within sampling variability.
- **Sensitivity:** RR 1.28–1.31 across weight truncation, overlap restriction and alternative PS models. E-value 1.88 (1.29 for the CI bound).
- **Interpretation:** an adjusted association consistent with higher 90-day GI-bleed risk for celecoxib, not a demonstrated causal effect. Key GI-risk confounders (PPI use, prior ulcer history) are unmeasured, the covariates are crude record counts, and the outcome is a recorded diagnosis.
- **Deviation from protocol:** full birth dates were available, so exact age at index was used instead of the index-year approximation.

## Methods audit

| Step | Method and key settings |
|---|---|
| Cohort extraction | DuckDB SQL over the raw OMOP CSVs; dates cast to `DATE`; windows with `BETWEEN` / `INTERVAL` |
| Propensity score | statsmodels logistic regression, main effects of 5 baseline covariates; no missing data |
| Weighting | Stabilized IPTW (P(T)/PS), untruncated in the primary analysis |
| Balance | Absolute SMD (pooled SD), before and after weighting; 0.10 threshold |
| Effect estimates | Weighted GLMs with robust HC0 SEs: Poisson log link (RR), Gaussian identity (RD) |
| Robustness | Weight truncation, overlap restriction, alternative PS models; bootstrap CI for D (1,000 resamples, seed 2026); E-value |

- **Method selection:** IPTW keeps every patient and, with stabilized weights, targets the ATE, which matches the estimand. Own code (DuckDB + statsmodels) was used instead of the listed reference packages because the design is a fixed 90-day contrast with five covariates: CohortMethod is R-based and built for full-vocabulary, large-scale covariates; TrialEmulation, DoubleML and grf address longitudinal or heterogeneous effects. CohortMethod's large-scale PS and negative-control calibration inspired the innovation proposal.
- **Verification:** concept occurrence and source-to-standard mapping checked against the data; baseline (index −365 to −1) and follow-up (index +1 to +90) windows exclude index-day events; attrition counts independently recomputed and matched; weights and balance checked; GLM RRs matched hand calculations; the notebook runs top to bottom from a fresh kernel.
- **Two plausible but misleading results:**
  1. *"Balance is excellent, so confounding is removed."* Balance covers only five crude record counts; detect by checking balance on specific clinical covariates (prior ulcer, aspirin, anticoagulants), using negative-control outcomes and reporting the E-value.
  2. *"D is small, so confounding bias is small."* D reflects measured covariates only and its bootstrap CI includes 0; detect by bootstrapping D, re-estimating it with richer covariate sets, and comparing with negative-control calibration.

## References (own reading)

| Source | Used for |
|---|---|
| Andrade C. [Understanding relative risk, odds ratio, and related terms: as simple as it can get](https://www.psychiatrist.com/jcp/understanding-relative-risk-odds-ratio-related-terms/). *J Clin Psychiatry*. 2015;76(7):e857–e861. doi:10.4088/JCP.15f10150 | Interpreting RR and RD, and why RR is preferred over the odds ratio |
| Austin PC. [Balance diagnostics for comparing the distribution of baseline covariates between treatment groups in propensity-score matched samples](https://pubmed.ncbi.nlm.nih.gov/19757444/). *Stat Med*. 2009;28(25):3083–3107. doi:10.1002/sim.3697 | SMDs as the balance diagnostic and the 0.10 threshold |
| Austin PC. [An introduction to propensity score methods for reducing the effects of confounding in observational studies](https://pmc.ncbi.nlm.nih.gov/articles/PMC3144483/). *Multivariate Behav Res*. 2011;46(3):399–424. doi:10.1080/00273171.2011.568786 | Propensity scores, IPTW and balance checking |
| De Paepe AL, Gibby A, Oporto-Lisboa L, et al. [Building causal models in pain research: the case of executive functioning and transitions in pain states](https://pmc.ncbi.nlm.nih.gov/articles/PMC12794360/). *Pain*. 2026;167(2):414–427. doi:10.1097/j.pain.0000000000003833 | Building the DAG; confounders vs mediators and colliders |
| Karim ME. [Understanding plasmode simulation](https://ehsanx.github.io/PlasmodeSim/). Online tutorial; 2022. | Plasmode simulation for validating the innovation proposal |
| OHDSI. [Standardized Data: The OMOP Common Data Model](https://www.ohdsi.org/data-standardization/) | OMOP CDM and standardized vocabularies (concept identification) |
| Shin S. [Safety of celecoxib versus traditional nonsteroidal anti-inflammatory drugs in older patients with arthritis](https://pmc.ncbi.nlm.nih.gov/articles/PMC6299466/). *J Pain Res*. 2018;11:3211–3219. doi:10.2147/JPR.S186000 | Real-world context; gastroprotective co-medication as an unmeasured confounder |

Method reference cited in the analysis: VanderWeele TJ, Ding P. Sensitivity analysis in observational research: introducing the E-value. *Ann Intern Med*. 2017;167(4):268–274.

## AI use disclosure

Claude (Anthropic) was used as a tutor and coding assistant throughout this assessment. It explained methods, gave hints, reviewed code I wrote.
