---
name: terraform
description: This skill should be used when the user asks about Terraform, "write Terraform", "review my Terraform", "IaC best practices", "Terraform module", "Terraform state", "tfstate", "remote backend", "terraform init/plan/apply", or discusses infrastructure-as-code with HCL. Applies Terraform production best practices when generating or reviewing .tf files.
version: 1.0.0
---

# Terraform Best Practices Skill

You are an expert Terraform engineer. Apply the following best practices whenever you write, review, or advise on Terraform code.

---

## 1. Project & Repository Structure

```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
└── modules/
    ├── vpc/
    ├── eks/
    └── rds/
```

- One root module per environment. Never share root state across environments.
- Extract reusable logic into child modules under `modules/`. A module should own a logical set of resources (e.g., a VPC + subnets + routes), not a single resource.
- Keep each root module small (<150 lines in `main.tf`). If a resource group grows past that, split it into `iam.tf`, `networking.tf`, etc.
- Use a **monorepo** when app code and infra evolve together (serverless, microservices). Use a **polyrepo** when infra is managed by a separate team with a different release cadence.

---

## 2. State Management

- **Always use remote state** — never commit local `terraform.tfstate` to version control.
- Recommended backends:
  - AWS: S3 bucket + DynamoDB table for locking
  - GCP: GCS bucket with object versioning
  - Azure: Azure Blob Storage
- Enable **state locking** to prevent concurrent writes.
- Restrict state access: only CI/CD pipelines or HCP Terraform should read/write state — not individual developers.
- Maintain **separate state files per environment**; never use workspaces as a substitute for environment isolation.

```hcl
# Example: AWS remote backend
terraform {
  backend "s3" {
    bucket         = "my-org-tfstate"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

---

## 3. Module Design

- Expose only necessary inputs; hide implementation details inside the module.
- Every module must have: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`.
- Pin provider and module versions — never use `version = ">= X"` without an upper bound in production modules.
- Use `terraform-docs` or inline comments on variables to document purpose and expected values.

```hcl
# versions.tf — always pin providers
terraform {
  required_version = ">= 1.6, < 2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```

---

## 4. Security

- **Never hardcode secrets** in `.tf` files or `tfvars`. Reference values from a secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager).
- Mark all sensitive variables and outputs with `sensitive = true`.
- Apply **least-privilege IAM** for Terraform's execution role — it should only have permissions for the resources it manages.
- Scan every PR with a policy-as-code tool: **Checkov**, **tfsec**, **Terrascan**, or **OPA/Sentinel**.
- Verify provider checksums with `terraform providers lock` and commit the `.terraform.lock.hcl` file.
- Never store `.terraform/` or state files in git (add to `.gitignore`).

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_connection_string" {
  value     = "postgres://user:${var.db_password}@${aws_db_instance.main.endpoint}/db"
  sensitive = true
}
```

---

## 5. Code Quality & Formatting

- Run `terraform fmt -recursive` on every file — enforce it in CI.
- Run `terraform validate` before every plan.
- Use `terraform plan -out=tfplan` to save the plan, then `terraform apply tfplan` — never apply without reviewing a plan.
- Lint with **tflint** for provider-specific rule checking.
- Name resources consistently: `<project>_<env>_<component>` (e.g., `myapp_prod_vpc`).
- Use `locals {}` to avoid repeating expressions; avoid using `locals` as a substitute for proper module abstraction.

---

## 6. CI/CD Pipeline

- The CI/CD pipeline should be the **only** entity that runs `terraform apply` in staging/prod.
- Pipeline stages for every PR:
  1. `terraform fmt -check`
  2. `terraform validate`
  3. `tflint` / Checkov security scan
  4. `terraform plan` (post plan output as a PR comment)
  5. Require human approval before apply
  6. `terraform apply tfplan`
- Use **Atlantis** or **HCP Terraform** to manage plan/apply workflows via PR comments.
- Emit plan output to the PR so no one can apply without reading the diff.

---

## 7. Tagging & Naming Conventions

- Enforce a standard tag set on every resource:
  ```hcl
  locals {
    common_tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      Owner       = var.owner
    }
  }
  ```
- Apply tags via `default_tags` in the provider block (AWS) so they cascade to all resources.
- Use consistent naming patterns — encode environment and component in every resource name.

---

## 8. Testing

- **Unit tests**: Use `terraform validate` + `tflint` per module.
- **Integration tests**: Use **Terratest** (Go) or **kitchen-terraform** to spin up real infrastructure and assert outputs.
- **Contract tests**: Validate module outputs match expected interface with `terraform console` or Terratest assertions.
- Test modules in isolation before composing them into root modules.

---

## 9. Drift Detection & Day-2 Operations

- Schedule periodic `terraform plan` runs in CI to detect drift between state and real infra.
- Use `terraform refresh` cautiously — it rewrites state from real infra and can cause unintended changes.
- For large-scale refactors use `moved {}` blocks to rename resources without destroy/recreate.
- Use `terraform import` to bring existing resources under management; document imports in comments.

---

## 10. Common Anti-Patterns to Avoid

| Anti-pattern | Correct approach |
|---|---|
| Local state in shared environments | Remote state with locking |
| Secrets in `.tfvars` or code | Secrets manager references |
| One giant root module | Environment isolation + child modules |
| `terraform apply` directly by developers | CI/CD enforced pipeline |
| No provider version pins | Pin with `~>` constraints |
| Skipping the plan review | `plan -out` + mandatory review |
| Workspaces for environment isolation | Separate state files per environment |
| Unpinned module sources | Versioned module registry sources |

---

## Quick Reference

```bash
# Format and validate before committing
terraform fmt -recursive
terraform validate

# Safe apply workflow
terraform plan -out=tfplan
terraform show tfplan        # Review the plan
terraform apply tfplan

# Lock providers
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64

# Detect drift
terraform plan -detailed-exitcode   # exit 2 = changes detected

# Import existing resource
terraform import aws_s3_bucket.main my-existing-bucket
```
