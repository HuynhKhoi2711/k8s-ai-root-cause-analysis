# ml/

> **Status:** No ML code, datasets, or model artifacts exist yet.  
> This README documents the planned ML pipeline design.

---

## Purpose

This directory will contain everything related to the machine learning pipeline: dataset preparation, feature engineering, model training, evaluation, and MLflow experiment tracking. The output of this pipeline is a trained model artifact that the RCA API loads to serve predictions.

---

## Incident Labels

The classifier will distinguish between the following root-cause categories:

| Label | Description |
|-------|-------------|
| `normal` | No incident; baseline healthy traffic |
| `database_slow` | Database query latency significantly elevated, causing upstream timeouts |
| `payment_failure` | Payment Service returning high error rates (5xx) |
| `cpu_stress` | Node or container CPU saturation |

Additional labels may be introduced in later phases if the team collects sufficient data.

---

## Data Collection Plan

Metrics will be collected from **controlled incident runs** performed against the EKS cluster using scripts in `scripts/`. Each run must:

1. Start from a stable baseline state.
2. Inject a single known fault (one label per run).
3. Record the `incident_run_id`, start time, end time, and label for the run.
4. Export a Prometheus metric snapshot (PromQL query result) for the run window.

Metric snapshots will be stored in S3 (not committed to Git — large datasets are excluded by `.gitignore`).

---

## Dataset Schema

The exact feature schema must be agreed upon by the team before implementation begins. The following fields are expected:

| Column | Type | Description |
|--------|------|-------------|
| `incident_run_id` | string | Unique identifier for the incident run (used for group-based train/test split) |
| `timestamp` | float | Unix timestamp of the observation window |
| `label` | string | Root-cause category (one of the four labels above) |
| `request_rate_rps` | float | Requests per second (averaged over the observation window) |
| `error_rate_pct` | float | Percentage of 5xx responses |
| `p99_latency_ms` | float | 99th-percentile request latency in milliseconds |
| `cpu_usage_pct` | float | Pod CPU usage as a percentage of the requested limit |
| `memory_usage_mb` | float | Pod memory working set in MB |
| `pod_restart_count` | int | Number of Pod restarts during the observation window |

**The schema must be finalized before any training code or API input schema is written.** Schema changes after the API is deployed require a coordinated update of both the model and the API.

---

## Train/Test Split Strategy

`incident_run_id` will be used as the group key for train/test splitting. This ensures that all observations from a single incident run appear in either the training set or the test set, but not both. This prevents data leakage caused by temporal correlation within a single run.

Use `sklearn.model_selection.GroupShuffleSplit` or `GroupKFold` for this purpose.

---

## Planned Model Pipeline

### Step 1 — Baseline
A threshold-based classifier that applies hand-tuned rules (e.g., "if `error_rate_pct > 20` → `payment_failure`"). This establishes a minimum performance floor before any ML model is trained.

### Step 2 — Random Forest
A `sklearn.ensemble.RandomForestClassifier` trained on the features above. Start with default hyperparameters; tune if the baseline evaluation shows room for improvement.

### Evaluation Metrics
All models must be evaluated on the held-out test set with:
- Per-class **Precision**, **Recall**, and **F1-score**.
- **Macro-averaged F1** as the primary summary metric.
- A **confusion matrix** to identify which classes are confused.

Results must be logged to MLflow.

---

## MLflow Integration

All training runs will be tracked with MLflow:

- **Parameters:** model type, hyperparameters, feature list, dataset version.
- **Metrics:** Macro-F1, per-class Precision/Recall.
- **Artifacts:** trained model (`.pkl` or MLflow model format), confusion matrix plot, feature importance chart.

The MLflow tracking server URI is configured via the `MLFLOW_TRACKING_URI` environment variable (see `.env.example`). Do not hardcode a URI in training scripts.

The best model will be registered in the **MLflow Model Registry** and promoted to the `Staging` or `Production` stage before the RCA API loads it.

---

## Important Constraints

- Do **not** commit trained model files (`.pkl`, `.joblib`, `.h5`) to Git. Store them in MLflow artifacts backed by S3.
- Do **not** commit raw or processed datasets to Git. Store them in the S3 data bucket.
- The `.gitignore` at the repository root excludes `ml/models/`, `ml/artifacts/`, `ml/data/`, `mlruns/`, and common ML artifact file extensions.

---

## Planned Subdirectory Structure

The following structure will be created when implementation begins. It is **not created now** because no source files exist yet.

```
ml/
├── data/               ← gitignored; raw and processed datasets stored in S3
├── notebooks/          ← exploratory analysis (committed without output cells)
├── src/
│   ├── features.py     ← feature engineering functions
│   ├── train.py        ← training script
│   └── evaluate.py     ← evaluation script
└── requirements.txt    ← ML-specific Python dependencies
```

---

## Current Implementation Status

No training code, datasets, or model artifacts exist yet. Implementation begins in **Phase 8** of the project roadmap (see root `README.md`).
