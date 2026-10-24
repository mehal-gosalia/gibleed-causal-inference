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
- Overlap (FULL OUTER JOIN of distinct ids):
  | membership | n |
  |---|---|
  | both | 2,694 |
  | obs_period_only | 2,649 |
  | person_only | 0 |
- The 2,649 obs-period-only ids have no PERSON row → no year of birth / sex, so they
  can't get covariates and drop out when the cohort is joined to PERSON.
- Checked: those 2,649 ids have **0 DRUG_EXPOSURE and 0 CONDITION_OCCURRENCE records**
  → empty observation rows only; no patients are lost by requiring a PERSON row.
- Tip learned: a notebook cell only displays its last expression — use separate cells
  or `display(...)` to show multiple query results.
