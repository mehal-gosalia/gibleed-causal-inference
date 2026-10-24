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

**Still to do:** cohort attrition (rules 1–4); arm sizes and event counts; analysis dataset; baseline table.

### Task 2 — Causal study design and DAG *(not started)*

### Task 3 — Propensity scores, IPTW and diagnostics *(not started)*

### Task 4 — Confounding distortion, robustness and innovation *(not started)*

### Task 5 — Scientific interpretation *(not started)*

## AI use disclosure

Claude (Anthropic) was used as a tutor and coding assistant during this assessment: explaining methods, giving hints and reviewing code that I wrote, and helping with repository setup. All methodological decisions, code and written interpretation are my own, and I can explain each of them.
