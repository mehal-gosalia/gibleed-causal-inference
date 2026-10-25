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

### Task 1 — Data exploration and cohort feasibility *(in progress)*

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

**Still to do:** outcome flag and covariates (rules 5–8); arm sizes and event counts; analysis dataset and data dictionary; baseline table.

### Task 2 — Causal study design and DAG *(not started)*

### Task 3 — Propensity scores, IPTW and diagnostics *(not started)*

### Task 4 — Confounding distortion, robustness and innovation *(not started)*

### Task 5 — Scientific interpretation *(not started)*

## AI use disclosure

Claude (Anthropic) was used as a tutor and coding assistant throughout this assessment. It explained methods, gave hints, reviewed code I wrote.