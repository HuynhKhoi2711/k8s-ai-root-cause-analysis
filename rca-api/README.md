# rca-api/

> **Status:** No API source code exists yet.  
> This README documents the planned FastAPI service design.

---

## Purpose

This directory will contain the **Root Cause Analysis API**: a FastAPI application that loads the trained ML model from MLflow and serves root-cause predictions over HTTP.

The API is the user-facing output of the ML pipeline. It receives a set of observability metrics, passes them through the loaded model, and returns the predicted incident root-cause category.

---

## Planned Endpoints

### `GET /health`

Health check endpoint. Returns `200 OK` when the service is running and the model is loaded.

```json
{
  "status": "healthy",
  "model_version": "<mlflow-run-id-or-registered-version>"
}
```

### `POST /predict`

Accept a metric snapshot and return a root-cause prediction.

**Request body (JSON):**

```json
{
  "request_rate_rps": 42.5,
  "error_rate_pct": 0.2,
  "p99_latency_ms": 210.0,
  "cpu_usage_pct": 35.0,
  "memory_usage_mb": 512.0,
  "pod_restart_count": 0
}
```

**Response body (JSON):**

```json
{
  "predicted_label": "normal",
  "probabilities": {
    "normal": 0.87,
    "database_slow": 0.05,
    "payment_failure": 0.04,
    "cpu_stress": 0.04
  }
}
```

The `probabilities` field is included when the model supports `predict_proba` (e.g., Random Forest). If the model does not support probability output, only `predicted_label` will be returned.

---

## Input / Feature Schema Compatibility

The API input schema must exactly match the feature schema used during model training (see `ml/README.md`). If the feature schema changes, both the model and the API must be updated together and redeployed in coordination.

The API will validate input using **Pydantic** (included with FastAPI). Any request that is missing required fields or contains out-of-range values will return `422 Unprocessable Entity`.

---

## Model Loading Strategy

At startup, the API will load the model from MLflow using the registered model name and stage:

```python
import mlflow.sklearn

model = mlflow.sklearn.load_model("models:/<model-name>/Production")
```

The MLflow tracking URI is read from the `MLFLOW_TRACKING_URI` environment variable. The model name and stage may also be configurable via environment variables.

The model artifact is loaded once at startup and held in memory for the lifetime of the process. The API does not support hot-reloading a new model version without a Pod restart.

---

## Planned Subdirectory Structure

The following structure will be created when implementation begins. It is **not created now** because no source files exist yet.

```
rca-api/
├── Dockerfile
├── main.py             ← FastAPI app, endpoint definitions
├── model.py            ← Model loading and inference logic
├── schemas.py          ← Pydantic input/output models
└── requirements.txt    ← API-specific Python dependencies
```

---

## Current Implementation Status

No source code, Dockerfile, or configuration files exist yet. Implementation begins in **Phase 10** of the project roadmap (see root `README.md`).
