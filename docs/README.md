# docs/

> **Status:** No documentation files have been created yet beyond the README files in each directory.  
> This README describes what will be documented here and when.

---

## Purpose

This directory will contain project-level documentation that does not belong in a single component's directory: architecture decision records, the agreed dataset schema, the experiment methodology, evaluation results, and demo instructions.

Documentation here must clearly distinguish between:
- **Design intent** — what the team plans to build.
- **Measured results** — what was actually observed, tested, or evaluated.

Do not create placeholder documents with no content. Documents will be added when there is real content to write.

---

## Planned Documents

| Document | Phase | Contents |
|----------|-------|----------|
| `architecture.md` | Phase 3+ | Detailed description of the provisioned AWS infrastructure, network topology, and EKS configuration. Written after Terraform apply. |
| `dataset-schema.md` | Phase 7 | Agreed-upon feature schema for the ML training dataset. This must be signed off by the team before any training code or API schema is written. |
| `experiment-methodology.md` | Phase 7 | Description of how controlled incidents are generated, what parameters are varied, how many runs are collected per label, and how data quality is validated. |
| `evaluation-results.md` | Phase 8–9 | Confusion matrices, Macro-F1 scores, per-class Precision/Recall, and comparison between the threshold baseline and Random Forest. |
| `demo-instructions.md` | Phase 12 | Step-by-step guide for running a live demonstration: starting the cluster, injecting an incident, calling the RCA API, and interpreting the result. |

---

## Documentation Standards

1. Use clear, factual language. Avoid vague claims.
2. Include dates or Git SHAs to anchor measurements to a specific code state.
3. Diagrams should be included as Mermaid code blocks (renders in GitHub) or as exported images committed alongside the Markdown file.
4. Do not include raw AWS account IDs, credentials, or IP addresses in committed documentation.

---

## Current Implementation Status

No documentation files exist yet beyond this README. Documents will be added incrementally as the corresponding project phases are completed (see root `README.md`).
