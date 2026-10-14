# kubernetes/

> **Status:** No Kubernetes manifests have been created yet.  
> This README documents what will be built and why.

---

## Purpose

This directory will contain the Kubernetes application manifests that deploy the microservices, monitoring stack components, and the RCA API onto the Amazon EKS cluster provisioned by Terraform.

Kubernetes manifests are **application workload configuration** and are separate from AWS infrastructure (which lives in `terraform/`). Terraform creates the EKS cluster; this directory defines what runs inside it.

---

## Planned Kubernetes Resources

For each microservice and the RCA API, the following resource types will be created:

| Resource | Purpose |
|----------|---------|
| `Namespace` | Isolate workloads (e.g., `rca-system`, `monitoring`) |
| `Deployment` | Declare desired Pod replicas, image, resource limits |
| `Service` | Stable DNS name and cluster-internal load balancing |
| `ConfigMap` | Non-sensitive runtime configuration (environment variables, config files) |
| `Secret` | Sensitive values (database passwords, API tokens) — see the secrets note below |
| `Ingress` | Route external HTTP traffic through the AWS Load Balancer Controller |
| `HorizontalPodAutoscaler` | (Optional) Auto-scale Pods based on CPU/memory during load tests |

The Prometheus and Grafana manifests will be managed by Helm (`kube-prometheus-stack`); their configuration files live in `monitoring/`.

---

## Secrets Handling

**Never commit plaintext secrets to Git.**

The planned approach for managing secrets in the MVP:

1. Use Kubernetes `Secret` objects for values that Pods must access at runtime.
2. Populate secrets via `kubectl create secret` or a secure CI/CD step — not by committing YAML files containing real values.
3. `.gitignore` is configured to exclude `*.env` and `*.tfvars` files.
4. Consider AWS Secrets Manager integration in a later phase if the team has capacity.

---

## Inspection Commands

The following commands will be useful once manifests are applied:

```bash
# List all Pods in the rca-system namespace
kubectl get pods -n rca-system

# Describe a specific Pod (useful for debugging startup failures)
kubectl describe pod <pod-name> -n rca-system

# Tail logs from a running container
kubectl logs -f deployment/<deployment-name> -n rca-system

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n rca-system

# Inspect a Service
kubectl get svc -n rca-system

# Get Ingress address (ALB DNS name)
kubectl get ingress -n rca-system
```

---

## Planned Subdirectory Structure

The following structure will be created when implementation begins. It is **not created now** because no manifest files exist yet.

```
kubernetes/
├── namespaces/
├── api-gateway/
├── order-service/
├── payment-service/
└── rca-api/
```

Each subdirectory will contain the Deployment, Service, ConfigMap, and (where needed) Ingress manifests for that workload.

---

## Current Implementation Status

No manifests exist yet. Implementation begins in **Phase 5** of the project roadmap (see root `README.md`).
