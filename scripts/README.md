# scripts/

> **Status:** No scripts exist yet.  
> This README documents the kinds of scripts that will be created and why.

---

## Purpose

This directory will contain operational and experiment support scripts. Scripts here assist with tasks that are too specific or stateful to belong in CI/CD pipelines, such as triggering controlled incidents, collecting labeled metric snapshots, or checking environment prerequisites.

Scripts must have a clear, single purpose and must include a usage comment at the top of the file. They must never unexpectedly delete AWS resources, destroy Kubernetes workloads, or modify production state without explicit confirmation.

---

## Planned Scripts

The following scripts are anticipated. They will be created during the relevant project phase and will not be stubbed out in advance.

| Script | Phase | Purpose |
|--------|-------|---------|
| `check-prerequisites.sh` | Phase 1 | Verify that required CLI tools (AWS CLI, Terraform, kubectl, Helm, Docker) are installed and at minimum versions |
| `bootstrap-state-bucket.sh` | Phase 2 | Create the S3 Terraform state bucket with versioning, encryption, and Block Public Access enabled |
| `generate-load.sh` | Phase 7 | Send controlled HTTP load to the API Gateway to produce baseline metric data |
| `inject-incident.sh` | Phase 7 | Trigger a specific incident type (e.g., CPU stress via a stress Pod, payment error via a feature flag) |
| `collect-metrics.sh` | Phase 7 | Query Prometheus for the required metrics over a specified time window and write a labeled snapshot to S3 |

---

## Script Standards

When scripts are written, they must follow these standards:

1. **Header comment** — Include the script name, purpose, usage example, and any prerequisites at the top.
2. **Fail fast** — Use `set -euo pipefail` in Bash scripts.
3. **No silent destructive actions** — Scripts that delete or modify AWS resources must print a clear warning and require explicit confirmation (e.g., `read -r confirm`).
4. **No hardcoded credentials** — Read AWS identity from the environment or an AWS CLI profile; never embed access keys.
5. **Idempotent where possible** — Running a script twice should not cause errors or duplicate resources.

---

## Current Implementation Status

No scripts exist yet. Scripts will be created incrementally starting in **Phase 2** (bootstrap) and **Phase 7** (incident generation) of the project roadmap (see root `README.md`).
