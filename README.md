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

## Findings by task

### Task 1 — Data exploration and cohort feasibility *(done)*

**Data exploration (done)**
- Relevant OMOP tables: PERSON, OBSERVATION_PERIOD, DRUG_EXPOSURE, CONDITION_OCCURRENCE, VISIT_OCCURRENCE, CONCEPT, CONCEPT_ANCESTOR. The key linking them is `person_id`.
- **PERSON:** 2,694 patients, each with year of birth and sex.
- **OBSERVATION_PERIOD:** 5,343 distinct `person_id`s, with **exactly one period per person**. Because there are no gaps between periods, the continuous-coverage rule (index −365 to +90) can be checked against a single period per patient.
- **PERSON vs OBSERVATION_PERIOD overlap:**

  | Membership | Patients |
  |---|---|
  | In both | 2,694 |
  | OBSERVATION_PERIOD only | 2,649 |
  | PERSON only | 0 |

- The 2,649 OBSERVATION_PERIOD-only IDs have no PERSON record (so no age or sex) and **0 drug-exposure and 0 condition records**. They are empty observation rows: requiring a PERSON record removes no patients who could enter the cohort.

**Concept sets (done)**

Concepts were identified with CONCEPT and CONCEPT_ANCESTOR (each starting concept plus all its descendants), and selection uses the standard `DRUG_CONCEPT_ID` / `CONDITION_CONCEPT_ID` fields.

| concept_id | Name | Role | Inclusion rule |
|---|---|---|---|
| 1118084 | celecoxib | Exposure (treatment = 1) | Ingredient + all CONCEPT_ANCESTOR descendants, via `DRUG_CONCEPT_ID` |
| 1124300 | Diclofenac | Exposure (comparator = 0) | Ingredient + all CONCEPT_ANCESTOR descendants, via `DRUG_CONCEPT_ID` |
| 192671 | Gastrointestinal hemorrhage (SNOMED) | Outcome and baseline exclusion | Standard concept + descendants, via `CONDITION_CONCEPT_ID` |
| 35208414 | Gastrointestinal hemorrhage, unspecified (ICD10CM) | Source code, mapping check only | Not used for selection; non-standard, "Maps to" 192671 |

- **Coverage in the data:** celecoxib 1,844 records / 1,844 patients; diclofenac 850 / 850; GI hemorrhage 479 records / 479 patients. Every record uses the starting concept itself; no descendant concepts occur.
- **Incomplete vocabulary (reported, as the brief requires):** the vocabulary snapshot is heavily trimmed (CONCEPT has 445 rows). The example formulations 1118088 (celecoxib 200 MG Oral Capsule [Celebrex]) and 40162359 (Diclofenac Sodium 75 MG Delayed Release Oral Tablet) exist in CONCEPT but have **no CONCEPT_ANCESTOR link** to their ingredients. This does not affect the cohort, because all study-drug exposures are recorded at the ingredient level.
- **Source mapping:** all 479 GI hemorrhage records come from ICD10CM source code 35208414, which maps to SNOMED 192671.
- **Exposure structure:** each patient has at most one record per study drug, and **0 patients have records for both drugs**. So the same-day dual-start exclusion removes nobody, and treatment switching between the two study drugs does not occur in this data.
- **Not included in the outcome:** "Peptic ulcer" (4027663, 802 records) is present but is not a bleeding diagnosis. It could be used for a broader-phenotype sensitivity analysis.

**Cohort design (done)**

| Component | Implementation |
|---|---|
| Design | New-user, active-comparator cohort; celecoxib = 1, diclofenac = 0 |
| Index date | Earliest celecoxib or diclofenac `DRUG_EXPOSURE_START_DATE`; one index episode per patient |
| Baseline | Index −365 to index −1 |
| Observation requirement | One OBSERVATION_PERIOD covering index −365 through index +90 (each patient has only one period, so no gaps can occur) |
| Exclusions | Same-day dual starts; any study-drug use in baseline (new-user rule); GI bleed (192671) in baseline |
| Follow-up | Index +1 to index +90; index-day outcomes are not counted |
| Exposure handling | Intention-to-treat style: group assigned at index and kept for 90 days regardless of switching or discontinuation |
| Censoring | None within follow-up: the observation requirement means everyone is fully observed for 90 days (fixed-window risk analysis) |

**Cohort attrition (rules 1–4)**

| Step | Remaining | Excluded |
|---|---|---|
| Patients with a celecoxib or diclofenac exposure | 2,694 | — |
| Exclude same-day dual starts | 2,694 | 0 |
| Exclude study-drug use in 365-day baseline (new users) | 2,694 | 0 |
| Continuous observation index −365 to +90 | 2,605 | 89 |
| Exclude GI bleed in 365-day baseline | 2,605 | 0 |
| **Final cohort** | **2,605** (celecoxib 1,781; diclofenac 824) | |

- **Multiple exposures, switching, both groups:** every patient has a single study-drug record and none has both drugs, so there are no repeat exposures, no same-day dual starts, and no crossover between arms (crossover frequency = 0).
- **New-user rule removes nobody** because the index date is, by definition, the earliest study-drug exposure. The rule is meaningful only together with the 365-day observed lookback, which confirms the patient was under observation during the washout window.
- **Observation requirement removes 89 patients** (2,694 → 2,605), all because their records end less than 90 days after index; everyone had the full 365-day lookback. Requiring complete follow-up can introduce selection bias if early loss to follow-up is related to treatment or prognosis.
- **No baseline GI bleeds:** no cohort member has a GI bleed recorded at any time before index, or on the index day itself. This is likely an artefact of how the synthetic data were generated, and real data would be expected to have some baseline exclusions.

**Analysis dataset (done)**

One row per patient (2,605 rows, all unique, no missing values):
`patient_id, index_date, treatment, gi_bleed_90d, age, sex, baseline_condition_count, baseline_drug_count, baseline_visit_count`.

| Arm | Patients | GI bleed events (90 days) | Risk |
|---|---|---|---|
| Celecoxib (1) | 1,781 | 343 | 19.3% |
| Diclofenac (0) | 824 | 121 | 14.7% |

Both arms have enough events for estimation (no sparse-event problem).

**Data dictionary (derived fields)**

| Field | Type | Source | Definition |
|---|---|---|---|
| `patient_id` | int | PERSON | `PERSON_ID`; one row per patient |
| `index_date` | date | DRUG_EXPOSURE | Earliest celecoxib or diclofenac `DRUG_EXPOSURE_START_DATE` |
| `treatment` | 0/1 | DRUG_EXPOSURE | 1 = celecoxib (1118084), 0 = diclofenac (1124300); fixed at index (ITT) |
| `gi_bleed_90d` | 0/1 | CONDITION_OCCURRENCE | 1 if any GI hemorrhage (192671) starts on index +1 to +90; counts patients, not diagnosis rows |
| `age` | int (years) | PERSON | Completed years from full birth date (year, month, day) to `index_date` |
| `sex` | M/F | PERSON | `GENDER_CONCEPT_ID` 8507 = M, 8532 = F |
| `baseline_condition_count` | int | CONDITION_OCCURRENCE | Distinct condition **records** starting index −365 to −1 (not unique diseases) |
| `baseline_drug_count` | int | DRUG_EXPOSURE | Distinct drug-exposure **records** starting index −365 to −1 (not unique ingredients) |
| `baseline_visit_count` | int | VISIT_OCCURRENCE | Distinct visit **records** starting index −365 to −1 |

- **Deviation from protocol (improvement):** the protocol allows age to be approximated as index year − birth year *if the full birth date is unavailable*. Month and day of birth are recorded for every patient, so exact age in completed years was used (mean 38.2 vs 38.6 with the approximation).
- **Crossover (rule 7 diagnostic):** 0 patients in either arm have a record of the other study drug during follow-up.
- **Sparse covariates:** age ranges only 31–46 years. Baseline condition count (median 0, max 5) and drug count (median 0, max 12) are mostly zero. **Baseline visit count is non-zero for only 4 of 2,605 patients**, so it carries almost no information about healthcare use and is a weak proxy for utilisation-related confounding.

**Baseline table by treatment (done)**

Mean (SD) for continuous and count variables; n (%) for male. Absolute standardized mean difference (SMD) = |difference in means| ÷ pooled SD; 0.10 is the reference threshold for imbalance.

| Covariate | Celecoxib (n = 1,781) | Diclofenac (n = 824) | Abs. SMD (unweighted) |
|---|---|---|---|
| Age, years | 38.17 (3.29) | 38.10 (3.25) | 0.022 |
| Male | 885 (49.7%) | 391 (47.5%) | 0.045 |
| Baseline condition count | 0.63 (0.83) | 0.45 (0.70) | **0.233** |
| Baseline drug count | 0.32 (0.72) | 0.35 (0.71) | 0.044 |
| Baseline visit count | 0.00 (0.04) | 0.00 (0.03) | 0.012 |

- Before weighting, **baseline condition count is the only imbalanced covariate** (SMD 0.233 > 0.10): celecoxib initiators had more recorded conditions in the year before index. This is consistent with confounding by indication, if COX-2-selective celecoxib is preferentially prescribed to patients judged to be at higher risk.
- The other covariates are already balanced (SMD < 0.05).

**Task 1 complete.**

### Task 2 — Causal study design and DAG *(done)*

- **Target trial question:** among adult new users of celecoxib or diclofenac with no use of either drug and no GI bleed in the prior year, does starting celecoxib instead of diclofenac change the 90-day risk of a recorded GI bleed?
- **Treatment strategies:** initiate celecoxib (1) vs initiate diclofenac (0) at index; intention-to-treat analogue (arm fixed at index for 90 days).
- **Estimand:** 90-day risk ratio (primary) and risk difference; average treatment effect (ATE) in the eligible cohort, estimated with stabilized IPTW.
- **Time zero:** eligibility, treatment assignment and start of follow-up all coincide at the index date, avoiding immortal time.
- **DAG** (in the notebook): measured confounders (age, sex, baseline condition/drug/visit counts) → treatment and outcome; **unmeasured confounders** (prior ulcer / GI-risk history; gastroprotective PPI/H2RA co-medication; arthritis severity and aspirin/anticoagulant use); a **mediator** (post-index dose/adherence) and a **collider** (healthcare visit during follow-up), neither of which is adjusted for.
- **Covariate justification:** all PS covariates are measured in the 365-day baseline, so none can be affected by treatment, and each plausibly affects both prescribing and bleeding risk. Post-index variables are excluded because they are mediators or colliders.

| Assumption | Assessment |
|---|---|
| Conditional exchangeability | Questionable: measured covariates balanced, but key GI risk factors are unmeasured (E-value 1.88) |
| Positivity | Supported: good PS overlap, 5 patients outside common support, max weight 2.82 |
| Consistency | Partly met: dose, formulation and duration not recorded |
| No selection bias | Possible threat: 89 patients (3.3%) excluded for incomplete follow-up |
| No outcome misclassification | Possible threat: recorded diagnosis, not adjudicated bleed |
| Correct PS specification | Reasonable: balance achieved; alternative specifications give the same RR |

### Task 3 — Propensity scores, IPTW and diagnostics *(done)*

**Propensity score model (done)**

- **Specification:** logistic regression of treatment (celecoxib = 1) on age, male, baseline condition count, baseline drug count and baseline visit count, with main effects only. All covariates are measured in the 365-day baseline; no post-index variables are included.
- **Missing data:** none; every covariate is complete for all 2,605 patients, so no imputation was needed.

| Covariate | Coefficient (log-odds) | 95% CI | p |
|---|---|---|---|
| Age (per year) | 0.013 | −0.012 to 0.039 | 0.307 |
| Male | 0.086 | −0.081 to 0.252 | 0.314 |
| Baseline condition count | **0.357** | 0.236 to 0.477 | <0.001 |
| Baseline drug count | **−0.178** | −0.300 to −0.055 | 0.004 |
| Baseline visit count | 0.149 | −2.133 to 2.431 | 0.898 |

- **Main driver of treatment choice:** each additional baseline condition record raises the odds of starting celecoxib by about 43% (OR ≈ 1.43), consistent with the imbalance in the baseline table. The visit-count estimate is very imprecise because only 4 patients have a baseline visit.
- **Discrimination:** c-statistic (AUC) = 0.57, so treatment choice is only weakly predictable from the measured covariates. This means good overlap, not absence of confounding, since unmeasured factors are not in the model.
- **Overlap and positivity:** PS ranged 0.31–0.90 in celecoxib and 0.53–0.89 in diclofenac, and the distributions largely coincide. Only **5 celecoxib patients** fall outside the diclofenac PS range (4 just above 0.888, 1 at 0.308). The distribution is spiky because the covariates are low-valued counts, giving about 290 distinct PS values.

**Stabilized IPTW and diagnostics (done)**

- **Weights:** celecoxib = P(T=1) / PS; diclofenac = P(T=0) / (1 − PS), with P(T=1) = 1,781 / 2,605 = 0.684.

| | Celecoxib | Diclofenac |
|---|---|---|
| Mean weight | 1.000 | 0.999 |
| Range | 0.76 – 2.22 | 0.67 – 2.82 |
| Effective sample size (actual n) | 1,768 (1,781) | 792 (824) |

- **Extreme weights:** none. The largest weight is 2.82 (a diclofenac patient with PS 0.888), and only 5 patients exceed 2.0. Truncation is not needed for the primary analysis, and is tested as a sensitivity analysis.
- **Effective sample size** stays at 96–99% of the actual group sizes, so weighting loses very little information.

| Covariate | Abs. SMD unweighted | Abs. SMD weighted |
|---|---|---|
| Age | 0.022 | 0.003 |
| Male | 0.045 | 0.001 |
| Baseline condition count | **0.233** | 0.002 |
| Baseline drug count | 0.044 | 0.009 |
| Baseline visit count | 0.012 | 0.004 |

- **Balance:** after weighting, every covariate has |SMD| < 0.01, well below the 0.10 threshold, including baseline condition count (0.233 → 0.002). A Love plot is in the notebook.
- **Limitation:** balance is achieved only on the measured covariates. Unmeasured confounders (for example GI risk factors, gastroprotective co-medication, NSAID dose and duration) are not addressed by weighting.

**Crude vs IPTW-adjusted effect estimates (done)**

| Analysis | Risk celecoxib | Risk diclofenac | RR (95% CI) | RD (95% CI) |
|---|---|---|---|---|
| Crude | 19.3% | 14.7% | 1.31 (1.08–1.59) | 4.6% (1.5–7.6%) |
| IPTW (stabilized) | 18.8% | 14.7% | 1.28 (1.05–1.55) | 4.1% (1.1–7.2%) |

- **Method:** RR from a Poisson GLM with log link and RD from a linear GLM, regressing the outcome on treatment with stabilized weights and robust (HC0) standard errors. Robust SEs do not account for estimation of the propensity score and are typically conservative for IPTW.
- **Interpretation:** after balancing measured covariates, celecoxib initiators still had about 28% higher 90-day risk of a recorded GI bleed (about 4 extra events per 100 patients). Adjustment moved the estimate only slightly, so measured confounding explains little of the crude association.
- **Robustness note:** a bootstrap that re-fits the propensity model failed in some resamples, because resamples without any of the 4 patients with a baseline visit make `baseline_visit_count` constant. This is a practical consequence of the near-constant covariate.

**Task 3 complete.**

### Task 4 — Confounding distortion, robustness and innovation *(in progress)*

**Confounding distortion (done)**

D = log(RR_crude) − log(RR_adjusted) = log(1.312) − log(1.280) = **0.024**, so the crude RR was about 2.5% higher than the IPTW-adjusted RR.

- **Sign:** positive. The measured covariates (mainly baseline condition count) confounded the crude comparison towards making celecoxib look riskier, consistent with sicker patients being channelled to celecoxib.
- **Magnitude:** small. Most of the crude association remains after adjustment.
- **Why D is not the true total confounding bias:** it reflects only the *measured* covariates. It cannot capture unmeasured confounders (for example gastroprotective co-medication, GI risk history, NSAID dose and duration, over-the-counter aspirin). The measured covariates are weak proxies (visit count is almost always zero). D also depends on correct PS-model specification and random error. A small D therefore means "adjustment for what we measured changed little", not "there is little confounding".

**Sensitivity analyses (done)**

| Analysis | n | RR (95% CI) | Max weight |
|---|---|---|---|
| Primary: stabilized IPTW, all covariates | 2,605 | 1.28 (1.05–1.55) | 2.82 |
| Weights truncated at 1st/99th percentile | 2,605 | 1.29 (1.06–1.56) | 1.55 |
| Weights truncated at 5th/95th percentile | 2,605 | 1.28 (1.06–1.55) | 1.19 |
| Overlap restriction (common PS support) | 2,600 | 1.28 (1.05–1.55) | 2.68 |
| Alternative PS: drop visit count | 2,605 | 1.28 (1.05–1.55) | 2.82 |
| Alternative PS: age + sex only | 2,605 | 1.31 (1.08–1.59) | 1.07 |

- The estimate is stable across weight truncation, overlap restriction and covariate sets, so the result is not driven by extreme weights, non-overlap or the near-constant visit covariate.
- Adjusting for age and sex alone reproduces the crude RR, which shows that baseline condition count accounts for essentially all of the adjustment.

**Unmeasured confounding: E-value (VanderWeele & Ding 2017)**

- E-value = **1.88** for the point estimate and **1.29** for the lower confidence limit.
- An unmeasured confounder associated with both celecoxib initiation and GI bleeding by a risk ratio of about 1.9 each (beyond the measured covariates) could fully explain the observed RR; one of about 1.3 could shift the confidence interval to include 1. Gastroprotective co-medication is a plausible candidate of this strength (Shin 2018), so the result should not be read as causal.

**Still to do:** research innovation proposal (250–400 words).

### Task 5 — Scientific interpretation *(not started)*

## References (own reading)

| Source | Used for |
|---|---|
| Austin PC. [Balance diagnostics for comparing the distribution of baseline covariates between treatment groups in propensity-score matched samples](https://pubmed.ncbi.nlm.nih.gov/19757444/). *Stat Med*. 2009;28(25):3083–3107. doi:10.1002/sim.3697 | Standardized mean differences as the recommended balance diagnostic (rather than significance tests), and the 0.10 threshold used for the baseline table and the before/after-weighting comparison (Tasks 1 and 3). |
| Austin PC. [An introduction to propensity score methods for reducing the effects of confounding in observational studies](https://pmc.ncbi.nlm.nih.gov/articles/PMC3144483/). *Multivariate Behav Res*. 2011;46(3):399–424. doi:10.1080/00273171.2011.568786 | Propensity score definition and assumptions; the four PS methods (matching, stratification, inverse probability of treatment weighting, covariate adjustment); building the PS model from baseline covariates and checking balance afterwards (Task 3). |
| De Paepe AL, Gibby A, Oporto-Lisboa L, et al. [Building causal models in pain research: the case of executive functioning and transitions in pain states](https://pmc.ncbi.nlm.nih.gov/articles/PMC12794360/). *Pain*. 2026;167(2):414–427. doi:10.1097/j.pain.0000000000003833 | A practical workflow for building a DAG from domain knowledge and prior studies, distinguishing confounders from mediators and colliders, and the gap between a conceptual DAG and what the available data can measure. Guided the DAG and covariate justification (Task 2). |
| Shin S. [Safety of celecoxib versus traditional nonsteroidal anti-inflammatory drugs in older patients with arthritis](https://pmc.ncbi.nlm.nih.gov/articles/PMC6299466/). *J Pain Res*. 2018;11:3211–3219. doi:10.2147/JPR.S186000 | Real-world context: a propensity-score-matched claims cohort comparing celecoxib with traditional NSAIDs. GI benefit appeared only with long use and disappeared without PPI/H2RA co-prescription, so gastroprotective co-medication is a plausible confounder not captured in this dataset. |

## AI use disclosure

Claude (Anthropic) was used as a tutor and coding assistant throughout this assessment. It explained methods, gave hints, reviewed code I wrote.