# Running notes — GiBleed causal inference

A log of what I did at each step, what I found, and decisions I made (with reasons).

---

## Setup (2026-10-23 / 24)
- conda env `gibleed`: Python 3.11.17, pandas 3.1.0, numpy 2.4.6, statsmodels 0.15.0,
  scikit-learn 1.9.1, matplotlib 3.11.2, duckdb 2.0.0, ipykernel 7.4.0
- Git repo initialised; data folder `GiBleed_5.5/` is gitignored (never committed).
- Pushed to private GitHub repo `mehal-gosalia/gibleed-causal-inference`.
- README.md added (package requirements + run instructions, as the brief requires).

## Step 1 — Load tables & explore
- Loaded the OMOP CSVs and queried them with DuckDB.
- OBSERVATION_PERIOD: 5,343 distinct person_ids, **exactly 1 period per person**
  → no gaps between periods, so rule 3 ("don't assume coverage across a gap") can't bite here.
- PERSON: 2,694 distinct person_ids; **all 2,694 also appear in OBSERVATION_PERIOD**.
- Overlap (FULL OUTER JOIN of distinct ids). This is the output that I got:
  ![alt text](image.png)
- The 2,649 obs-period-only ids have no PERSON row → no year of birth / sex, so they
  can't get covariates and drop out when the cohort is joined to PERSON.
- TODO: check whether those ids have any DRUG_EXPOSURE / CONDITION_OCCURRENCE records: ____

- In simple words, 
PERSON is the list of patients, with their age and sex. It has 2,694 people.
OBSERVATION_PERIOD records when each patient was being tracked in the database. It has 5,343 IDs.

- What results I got:
Every patient in PERSON also has a tracking period. Good.
Each person has only one tracking period, so there are no gaps to worry about.
2,649 IDs have a tracking period but no PERSON record, so we don’t know their age or sex. They’ll drop out naturally when you build the cohort.