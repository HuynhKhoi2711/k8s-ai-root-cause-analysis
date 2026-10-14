# terraform/

> **Status:** No Terraform configuration files have been created yet.  
> This README documents the planned AWS infrastructure and the S3 remote-state design.  
> No AWS resources have been provisioned; no `terraform init` or `terraform apply` has been run.

---

## Table of Contents

1. [Purpose](#purpose)
2. [Planned AWS Infrastructure](#planned-aws-infrastructure)
3. [Infrastructure vs. Kubernetes Workloads](#infrastructure-vs-kubernetes-workloads)
4. [Remote State with Amazon S3](#remote-state-with-amazon-s3)
5. [Bootstrap Sequence](#bootstrap-sequence)
6. [Backend Configuration](#backend-configuration)
7. [Migrating from Local State to S3](#migrating-from-local-state-to-s3)
8. [Terraform Workflow](#terraform-workflow)
9. [State Security and Access Control](#state-security-and-access-control)
10. [Sensitive Data Handling](#sensitive-data-handling)
11. [AWS Cost Risks](#aws-cost-risks)
12. [Planned Subdirectory Structure](#planned-subdirectory-structure)

---

## Purpose

Terraform provisions and manages the AWS infrastructure on which the Kubernetes workloads run. It is the single source of truth for the state of all AWS resources. All infrastructure changes must go through Terraform; do not make manual changes in the AWS Console without reflecting them in Terraform.

---

## Planned AWS Infrastructure

The following AWS resources are planned for the MVP. None exist yet.

### Networking

| Resource | Configuration |
|----------|--------------|
| **VPC** | Single VPC, CIDR block configurable (e.g., `10.0.0.0/16`) |
| **Public subnets** | Two subnets across two Availability Zones (e.g., `ap-southeast-1a`, `ap-southeast-1b`) |
| **Private subnets** | Two subnets across the same two AZs |
| **Internet Gateway** | Attached to the VPC; routes egress for public subnets |
| **NAT Gateway** | One NAT Gateway in a public subnet; provides internet egress for private subnets (required because EKS nodes in private subnets need to pull container images from ECR and communicate with the EKS API endpoint) |
| **Route tables** | Separate public and private route tables; public subnets route `0.0.0.0/0` to the IGW, private subnets route to the NAT Gateway |

### Compute and Orchestration

| Resource | Configuration |
|----------|--------------|
| **Amazon EKS cluster** | Kubernetes ≥ 1.29; API endpoint access set to private + public (restricted) |
| **EKS managed node group** | Instance type configurable (start with `t3.medium`); node count configurable |
| **IAM role for EKS cluster** | Grants EKS the permissions it needs to manage AWS resources on behalf of the cluster |
| **IAM role for EKS node group** | Grants EC2 nodes the permissions they need (ECR pull, CloudWatch logging, etc.) |

### Container Registry

| Resource | Configuration |
|----------|--------------|
| **Amazon ECR repositories** | One repository per service (api-gateway, order-service, payment-service, rca-api); image scanning enabled |

### EKS Add-ons

| Add-on | Purpose |
|--------|---------|
| **CoreDNS** | DNS resolution within the cluster |
| **kube-proxy** | Handles networking rules for Services |
| **Amazon VPC CNI** | Assigns VPC IP addresses to Pods |
| **AWS EBS CSI Driver** | Allows Kubernetes PersistentVolumeClaims to use EBS volumes (required for Prometheus persistent storage) |

### Additional Controllers

| Controller | Purpose |
|-----------|---------|
| **AWS Load Balancer Controller** | Provisions an ALB when a Kubernetes `Ingress` resource is created; requires an IAM role via IRSA |

---

## Infrastructure vs. Kubernetes Workloads

Terraform manages **AWS infrastructure** only. It provisions the EKS cluster, node group, networking, and IAM resources, but it does not deploy application workloads, Helm charts, or Kubernetes manifests.

| Terraform manages | Kubernetes manifests manage |
|-------------------|----------------------------|
| VPC, subnets, IGW, NAT Gateway | Namespace, Deployment, Service |
| EKS cluster and node group | ConfigMap, Secret, Ingress |
| IAM roles and policies | HorizontalPodAutoscaler |
| ECR repositories | ServiceMonitor (Prometheus) |
| EBS CSI Driver add-on | Helm releases (Prometheus, Grafana) |

This separation is intentional. It allows the infrastructure to be recreated without affecting application deployment logic, and vice versa.

---

## Remote State with Amazon S3

### Why remote state?

By default, Terraform stores state in a local `terraform.tfstate` file. This is unsuitable for a team environment because:

- The state file contains sensitive resource identifiers and outputs.
- Simultaneous `terraform apply` runs from different machines can corrupt the state.
- The state file must not be committed to Git.

The solution is to store state remotely in an **Amazon S3 bucket** with **state locking** to prevent concurrent modifications.

### S3 bucket requirements

The state bucket must be created **before** the main Terraform backend is initialised. It must have the following properties:

| Property | Requirement |
|----------|------------|
| **S3 Block Public Access** | All four Block Public Access settings must be enabled |
| **Bucket versioning** | Enabled; allows rollback to a previous state version if the current state becomes corrupted |
| **Encryption at rest** | SSE-S3 (`AES256`) is sufficient for the MVP; upgrade to SSE-KMS if the team requires a customer-managed key |
| **Access control** | Bucket policy must grant access only to the IAM identities that run Terraform (principle of least privilege) |
| **Bucket name** | Must be globally unique across all AWS accounts; choose a name that includes the project name and a random suffix, e.g., `ai-rca-tfstate-<random-suffix>` |

### State key

Use an explicit, descriptive state key to avoid collisions if the bucket is reused for multiple projects or environments:

```
ai-root-cause-analysis/dev/terraform.tfstate
```

### State locking

Terraform ≥ 1.10 supports **native S3 state locking** without requiring a DynamoDB table. Enable it with:

```hcl
use_lockfile = true
```

This causes Terraform to create a `.tflock` object in S3 during an operation and delete it when the operation completes. If a lock already exists when a new operation starts, Terraform will fail rather than corrupt the state.

> **Do not add a DynamoDB lock table.** The S3 native locking mechanism is sufficient for the MVP and avoids the additional cost and complexity of a DynamoDB table.

---

## Bootstrap Sequence

The S3 state bucket **cannot** be created by the same Terraform configuration that immediately uses it as the backend. This is a fundamental chicken-and-egg dependency: the backend bucket must exist before `terraform init` can connect to it.

The bootstrap process is therefore a manual one-time step that precedes all other Terraform work:

### Option A — Manual AWS CLI bootstrap (recommended for MVP)

```bash
# 1. Choose a unique bucket name and region
BUCKET_NAME="ai-rca-tfstate-<your-random-suffix>"
REGION="ap-southeast-1"

# 2. Create the bucket (ap-southeast-1 requires LocationConstraint)
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

# 3. Enable Block Public Access
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# 4. Enable versioning
aws s3api put-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --versioning-configuration Status=Enabled

# 5. Enable SSE-S3 encryption
aws s3api put-bucket-encryption \
  --bucket "$BUCKET_NAME" \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

A helper script (`scripts/bootstrap-state-bucket.sh`) that wraps these commands will be created in **Phase 2**. See `scripts/README.md`.

### Option B — Separate bootstrap Terraform configuration

A small, separate Terraform configuration (with its own **local** backend) can create the state bucket. After applying it, the main Terraform configuration can then be configured to use the bucket as its backend. This approach is cleaner for teams that want everything managed by Terraform, at the cost of maintaining two separate Terraform roots.

For the MVP, **Option A** is recommended because it is simpler and does not require managing a bootstrap state file.

---

## Backend Configuration

After the bucket has been created, configure the main Terraform backend as follows. This is documentation only — no `backend.tf` file has been created yet.

```hcl
terraform {
  backend "s3" {
    bucket       = "<globally-unique-state-bucket-name>"
    key          = "ai-root-cause-analysis/dev/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

**Important constraints on backend blocks:**

- The `backend` block does **not** support ordinary Terraform input variables (e.g., `var.bucket_name`). The values must be literals or supplied at `terraform init` time using `-backend-config` flags.
- Do not hardcode AWS access keys or secret keys in the backend block. Terraform reads AWS credentials from the environment (`AWS_PROFILE`, `AWS_DEFAULT_REGION`, instance profile, or the AWS credentials file).
- Replace `<globally-unique-state-bucket-name>` with the actual bucket name provisioned in the bootstrap step.

### Initialising with `-backend-config` flags (avoids hardcoding in source)

```bash
terraform init \
  -backend-config="bucket=ai-rca-tfstate-<your-suffix>" \
  -backend-config="key=ai-root-cause-analysis/dev/terraform.tfstate" \
  -backend-config="region=ap-southeast-1" \
  -backend-config="encrypt=true" \
  -backend-config="use_lockfile=true"
```

Using `-backend-config` flags keeps the bucket name out of the committed source code, which is useful when the bucket name is environment-specific or considered semi-sensitive.

Alternatively, a `backend.hcl` file (excluded from Git via `.gitignore` using `*.tfvars` / `*.hcl` patterns, but ensure this file specifically is excluded if needed) can supply these values. However, for simplicity, the team may choose to commit a `backend.tf` with the literal bucket name — the bucket name is not a secret.

---

## Migrating from Local State to S3

If any team member has already run `terraform apply` with local state and needs to migrate to S3:

1. **Back up the local state file first:**
   ```bash
   cp terraform.tfstate terraform.tfstate.backup-$(date +%Y%m%d)
   ```

2. **Add or update the `backend "s3"` block** in the Terraform configuration.

3. **Run `terraform init`** and respond `yes` when prompted to copy the existing state to the new backend:
   ```
   Do you want to copy existing state to the new backend?
     Enter a value: yes
   ```
   Terraform will upload the local state to the S3 key specified in the backend configuration.

4. **Verify the migration** by running `terraform plan` and confirming no unexpected changes are shown.

5. **Delete the local state file** only after confirming the S3 state is correct:
   ```bash
   rm terraform.tfstate
   ```
   The backup file (`terraform.tfstate.backup-<date>`) can be retained temporarily and then deleted once the team is confident in the migrated state.

---

## Terraform Workflow

The standard workflow once the backend is initialised:

```bash
# Format all .tf files (check only; exit non-zero if formatting is needed)
terraform fmt -check

# Initialise the working directory and connect to the S3 backend
terraform init

# Validate the configuration syntax and references
terraform validate

# Show the planned changes before applying
terraform plan

# Apply the planned changes (requires explicit approval)
terraform apply

# Destroy all managed resources (use with caution; see note below)
terraform destroy
```

### Note on `terraform destroy` and the state bucket

`terraform destroy` on the **main** Terraform configuration destroys the AWS infrastructure (VPC, EKS, ECR, etc.) but does **not** destroy the S3 state bucket. The state bucket is not managed by the main Terraform configuration (it was created in the bootstrap step). This is intentional: destroying the state bucket before all managed resources are cleanly destroyed could make it impossible to recover the Terraform state.

Only delete the state bucket manually after:
1. All managed infrastructure has been destroyed.
2. The team has confirmed they will not continue using this project.

### `.terraform.lock.hcl`

The `.terraform.lock.hcl` file records the provider versions selected by `terraform init`. **This file must be committed to Git** after the first `terraform init` so that all team members use the same provider versions. It is not excluded by `.gitignore`.

---

## State Security and Access Control

- **Never commit `.tfstate` files to Git.** The `.gitignore` at the repository root excludes all `*.tfstate` and `*.tfstate.*` files.
- The S3 state bucket must have a bucket policy that restricts access to the IAM identities used by the team and by the Jenkins agent.
- Enable **S3 Object Lock** or rely on versioning to prevent accidental state deletion.
- The Jenkins IAM role must have `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` (for lock files), and `s3:ListBucket` on the state bucket, and nothing more.

---

## Sensitive Data Handling

Terraform state files can contain sensitive resource attributes (IAM key IDs, RDS passwords, etc.) in plaintext. To reduce risk:

- Use S3 server-side encryption (`encrypt = true` in the backend block).
- Do not output sensitive values from Terraform unless explicitly needed.
- Use `sensitive = true` on any output that contains a secret.
- Do not hardcode secrets in `.tf` files or `.tfvars` files. Use AWS Secrets Manager or environment variables instead.

---

## AWS Cost Risks

The following resources incur the largest ongoing costs:

| Resource | Risk |
|----------|------|
| **EKS cluster control plane** | Billed per hour even when no nodes are running (~USD 0.10/hour) |
| **EC2 node group** | Billed per EC2 instance hour; choose the smallest instance type that runs the workload |
| **NAT Gateway** | Billed per hour and per GB of processed data; one NAT Gateway is enough for the MVP (do not deploy one per AZ) |
| **ALB** | Billed per hour plus per LCU (Load Capacity Unit) |

**Destroy the EKS cluster and node group when not actively working.** The state bucket and ECR repositories can be left running at negligible cost.

---

## Planned Subdirectory Structure

The following structure will be created when Terraform configuration is authored. It is **not created now** because no `.tf` files exist yet.

```
terraform/
├── main.tf             ← Root configuration: VPC, EKS, ECR modules
├── variables.tf        ← Input variable declarations
├── outputs.tf          ← Output value declarations
├── versions.tf         ← Required providers and Terraform version constraints
├── backend.tf          ← S3 backend configuration (committed without real secrets)
└── .terraform.lock.hcl ← Provider lock file (committed after terraform init)
```

Terraform modules (e.g., a VPC module or EKS module) will be introduced only if the root configuration becomes unwieldy. Premature modularisation adds complexity without benefit at this stage.

---

## Current Implementation Status

No Terraform configuration files exist. Implementation begins in **Phase 3** of the project roadmap (see root `README.md`), after the S3 state bucket has been bootstrapped in **Phase 2**.
