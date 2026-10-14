# jenkins/

> **Status:** No Jenkinsfile or pipeline configuration exists yet.  
> This README documents the planned CI/CD pipeline design.

---

## Purpose

This directory will contain the Jenkins pipeline configuration (Jenkinsfile) that automates building, testing, and deploying the microservices and the RCA API to Amazon EKS.

Jenkins is the CI/CD engine for this project. It is triggered by Git pushes and runs the pipeline against the target EKS cluster.

---

## Planned Pipeline Stages

Each service (API Gateway, Order Service, Payment Service, RCA API) will have a pipeline with the following stages:

| Stage | Action |
|-------|--------|
| **Checkout** | Clone the repository at the triggering commit |
| **Test** | Run unit tests (e.g., `pytest`) and linting |
| **Build** | Build the Docker image, tagging it with the Git short SHA |
| **Push** | Authenticate to Amazon ECR and push the image |
| **Deploy** | Update the Kubernetes Deployment with the new image tag using `kubectl set image` or `helm upgrade` |
| **Smoke test** | Send a health-check request to the deployed service and verify a `200 OK` response |

The ML training and evaluation pipeline (Phases 8–9) may be added as an additional Jenkins job after the application deployment pipeline is stable.

---

## Credential Management

**Never hardcode AWS credentials, Kubernetes tokens, or API keys in a Jenkinsfile or in this repository.**

The planned approach:

- Store the AWS IAM credentials for ECR access in **Jenkins Credentials** (type: AWS Credentials).
- Store the `kubeconfig` for EKS access in **Jenkins Credentials** (type: Secret File).
- Reference credentials in the Jenkinsfile using the `withCredentials` block or the `credentials()` binding.
- Use IAM roles for service accounts (IRSA) where the Jenkins agent itself runs on EKS, so that no long-lived credentials are needed.

---

## Jenkins Agent Requirements

Jenkins agents must have the following tools installed:

- Docker (to build and push images)
- `kubectl` and `helm` (to deploy to EKS)
- AWS CLI v2 (to authenticate to ECR)
- Python ≥ 3.11 and `pytest` (to run tests)

If the Jenkins agent runs on EKS (recommended), these tools should be packaged into a custom Jenkins agent Docker image.

---

## Planned Subdirectory Structure

The following files will be created when implementation begins. They are **not created now** because no pipeline configuration exists yet.

```
jenkins/
└── Jenkinsfile         ← Declarative pipeline definition
```

A shared library or additional `Jenkinsfile` per service may be introduced later if the team decides to split pipelines.

---

## Current Implementation Status

No Jenkinsfile or pipeline configuration exists yet. Implementation begins in **Phase 11** of the project roadmap (see root `README.md`).
