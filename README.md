# Cox Survival Feature Selection on Microsoft Fabric — Making a Slow Pipeline Fast

## What this README covers
This README explains the companion notebook **`cox_feature_selection_fabric.ipynb`**. The
notebook takes a slow, production‑style **Cox survival feature‑selection** pipeline and
re‑engineers how it runs so it finishes **~5–6× faster while producing the exact same
model and accuracy**. Based on the notebook, we walk through:

- the **data** and where it comes from,
- the **model** we train and how we validate it,
- the **four execution strategies** for the feature‑selection search — **1 native baseline + 3 new optimizations**,
- the measured **time savings** and the **C‑index** (to prove speed improved while accuracy stayed identical),
- all tracked in **MLflow** on Microsoft Fabric.

---

## The data
- **Source:** a public **employee‑attrition** dataset (IBM HR Analytics Employee Attrition).
- **Landing:** the public file was **copied into blob / OneLake**, written as a managed
  **Delta table** (`cox_attrition_demo`), and then **read back with Spark** inside the notebook.
- **Size:** **1,470 rows × 32 columns**.
- **Candidate predictors:** **28** feature columns (tenure, satisfaction, job role,
  compensation, overtime, etc.).
- **Dependent (target) columns — the survival outcome:**

  | Column | Role | Meaning |
  |---|---|---|
  | `SURVIVAL_TIME_DAYS` | **duration** | how long the employee was observed |
  | `event` | **event indicator** | `1` = employee left / separated, `0` = still employed (censored) |
  | `FACILITY_NAME` | strata | groups the baseline hazard by site |
  | `PERSON_KEY` | id | row identifier (not a predictor) |

---

## The model
- **Algorithm:** **Cox Proportional Hazards** (`lifelines.CoxPHFitter`) — penalized and
  **stratified by facility**.
- **Feature selection:** greedy **best‑candidate forward selection** — each round fits every
  remaining candidate against the current set and admits the single best one only if it
  improves the score by a minimum threshold.
- **Validation metric:** **C‑index (concordance index)** — the survival‑analysis analog of
  ROC‑AUC, computed on a held‑out split (higher = better risk ranking; ~0.5 = random, 1.0 = perfect).

---

## The four execution strategies

![Cox Feature Selection Strategies in Fabric MLflow](Cox%20Feature%20Selection%20Strategies%20in%20Fabric%20MLflow.png)

The feature‑selection **search is identical** in all four — they differ only in **how/where the
per‑candidate model fits run**:

| # | Strategy | Type | How it works |
|---|---|---|---|
| 1 | **NAIVE** | **Native (baseline)** | Re‑reads from Spark (`count` + `toPandas`) **and** re‑encodes (one‑hot) on **every** candidate fit. |
| 2 | **CACHED (sequential)** | New | Extract **once** + encode **once** into an in‑memory design matrix; fit candidates one at a time on the driver. |
| 3 | **CACHED + multithread (driver)** | New | Same cache, but fit candidates **concurrently across the driver's CPU cores** (thread pool). |
| 4 | **CACHED + Spark executors** | New | **Broadcast** the cached matrix and fit candidates **in parallel across Spark executors**. |

> The single change that makes everything faster: stop re‑extracting from Spark and
> re‑encoding on every candidate. Do it **once**, then reuse the in‑memory matrix.

---

## Results — time saved (same run, same data)
All four strategies select the **identical** features and produce the **identical** C‑index.
Only the wall‑clock time changes.

| Strategy | Type | Wall time | Speedup vs native | Time saved |
|---|---|---|---|---|
| NAIVE | **Native baseline** | ~98 s (1 m 38 s) | 1.0× | — |
| CACHED (sequential) | New | ~19 s | **5.2×** | ~79 s |
| CACHED + multithread (driver) | New | ~21 s | **4.7×** | ~77 s |
| CACHED + Spark executors | New | ~15 s | **6.5×** | ~83 s |

---

## Results — accuracy unchanged (C‑index)

| Strategy | C‑index | Selected features |
|---|---|---|
| NAIVE | **0.9299** | YearsInCurrentRole, YearsWithCurrManager, JobSatisfaction, Education |
| CACHED (sequential) | **0.9299** | identical |
| CACHED + multithread (driver) | **0.9299** | identical |
| CACHED + Spark executors | **0.9299** | identical |

**Same model, same C‑index — ~5–6× faster.** These are pure execution optimizations, not
model changes: the result is bit‑for‑bit the same, only quicker.

---

## Notes
- Numbers are from a Fabric run on the 1,470‑row dataset. On data this small the three
  optimizations land close together (the Spark‑executor scheduling overhead offsets its
  parallelism, and exact ordering varies with cluster warmth). On **larger data / more
  candidates / a bigger cluster**, the driver‑thread and Spark‑executor strategies pull
  further ahead of the sequential one.
- Every strategy is logged to **MLflow** (experiment `cox_feature_selection_demo`) with
  parameters, per‑round train/validation C‑index, number of Cox fits, and timings — open the
  MLflow experiment to compare the runs side by side.

---

## Files in this folder
- `cox_feature_selection_fabric.ipynb` — the notebook.
- `Cox Feature Selection Strategies in Fabric MLflow.png` — the flow diagram shown above.
- `README.md` — this file.
