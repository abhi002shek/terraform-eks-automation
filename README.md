# terraform-eks-automation

Terraform automation to provision a full 3-tier application stack on AWS — VPC, ECR, and EKS — with Docker image builds and Kubernetes deployments handled end-to-end.

**Architectural Diagram:**

<img width="1280" height="720" alt="aws-eks-arch" src="https://github.com/user-attachments/assets/45bbb355-fe28-4187-b2ba-86da7ea0a1e2" />

---

## What This Does

A single Terraform stack that:

- **VPC** — Module-based VPC with public/private subnets, NAT gateway, and route tables across 2 AZs
- **ECR** — Creates ECR repositories, builds Docker images from local Dockerfiles, tags and pushes them automatically
- **EKS** — Provisions an EKS cluster with a managed node group, pulls images from ECR, and deploys them as pods
- **K8s Manifests** — Auto-generates Kubernetes manifests from templates with the correct ECR image URLs at apply time

Resource names include an environment suffix so the same code works for dev/stage/prod by swapping `.tfvars` files.

---

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) — configured with a profile (`aws configure --profile <name>`)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Docker (running locally)

---

## Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/abhi002shek/terraform-eks-automation.git
cd terraform-eks-automation
```

### 2. Set your AWS profile

Edit `dev.tfvars` and replace `your-aws-profile` with your actual AWS CLI profile name:

```hcl
aws_profile = "your-aws-profile"
```

### 3. (Optional) Remote state backend

If you want Terraform state stored in S3 with DynamoDB locking:

```bash
# Create S3 bucket
aws s3api create-bucket --bucket tf-state-<your-suffix> --region ap-south-2

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name tf-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
  --region us-east-1
```

Then configure the backend block in `providers.tf` accordingly.

### 4. Deploy

```bash
terraform init
terraform plan -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars -auto-approve
```

### 5. Verify

From the AWS Console, confirm VPC, ECR repositories, and EKS cluster are created.

From the CLI:

```bash
kubectl get nodes
kubectl get pods -n three-tier
```

---

## Destroy

```bash
terraform destroy -var-file=dev.tfvars -auto-approve
```

---

## Project Structure

```
.
├── main.tf              # VPC module + EKS cluster + node group
├── ecr.tf               # ECR repos, Docker build & push, manifest generation
├── iam.tf               # IAM roles for EKS cluster and node group
├── k8s.tf               # kubectl apply via local-exec
├── data.tf              # EKS cluster auth data sources
├── locals.tf            # Common tags
├── variables.tf         # Input variable declarations
├── outputs.tf           # ECR URLs output
├── providers.tf         # AWS + Kubernetes provider config
├── dev.tfvars           # Dev environment variable values
├── backend/             # Python Flask backend app + Dockerfile
├── frontend/            # Nginx frontend + Dockerfile
├── k8s/                 # Generated Kubernetes manifests (auto-created by Terraform)
└── k8s-templates/       # Manifest templates with image URL placeholders
```

> **Note on `.gitignore`:** The `k8s/` manifests and `terraform.tfstate` are committed here so the repo is fully self-contained for reference. In a real production workflow, add them to `.gitignore` and use a remote backend. See `.gitignore` for the commented-out lines ready to enable.
