# Terraform file templates

Canonical templates matching Loop's conventions. Substitute the `{{ MUSTACHE }}` placeholders
before writing files. Do not invent values — leave a placeholder or `TODO` if unknown.

## Model: one repo per project, environment is the single input

- **One repo per project.** A single Terraform root module lives in `terraform/`.
- **Environments are git branches** (`dev`, `prod`). The `.tf` code is identical across branches.
- **`environment` is the only environment input.** Everything else — project ID, the
  `tf-iac-sa` email, the state bucket — is **derived in `locals`** from the naming convention.
  No per-env `.tfvars` or backend files to keep in sync.
- **The one unavoidable external value:** a `backend` block is parsed before variables/locals
  exist, so it cannot read `var.environment`. The **state bucket** (and its impersonation SA)
  is therefore passed to `terraform init` via `-backend-config`, computed from the environment.
  The static `prefix` stays in the block.
- **`gcp-landingzone` owns base infra.** The GCP projects, buckets, and `tf-iac-sa` accounts
  already exist. This scaffold consumes them — never create projects/buckets/SAs here.

## Placeholders

| Placeholder | Meaning | Example |
| --- | --- | --- |
| `{{ ORG }}` | Org short name | `loop` |
| `{{ GROUP_ABBREV }}` | Group / domain abbrev | `pla` (platform) |
| `{{ PROJECT_NAME }}` | Project short name | `mcp` |
| `{{ PROJECT_DESCRIPTION }}` | One-line purpose | `manages the Drive + Claude MCP integration` |

Derived per environment: project ID `{{ ORG }}-{{ GROUP_ABBREV }}-<environment>-{{ PROJECT_NAME }}`,
bucket `<project-id>-iac-tfstate`, SA `tf-iac-sa@<project-id>.iam.gserviceaccount.com`.

## File layout (simple tier)

```
terraform/
├── providers.tf      # version + provider pins + provider blocks (use locals)
├── backend.tf        # partial: prefix static; bucket + SA via -backend-config
├── variables.tf      # environment (required), region, labels, enabled_apis
├── locals.tf         # naming convention -> project_id, tf SA, bucket, labels
├── main.tf           # google_project_service + resources
└── outputs.tf
```

## locals.tf (the single source of truth for naming)

```hcl
locals {
  # Fixed per repo (one repo per project)
  org     = "{{ ORG }}"
  group   = "{{ GROUP_ABBREV }}"
  project = "{{ PROJECT_NAME }}"

  # Derived from the environment input
  project_id                = "${local.org}-${local.group}-${var.environment}-${local.project}"
  terraform_service_account = "tf-iac-sa@${local.project_id}.iam.gserviceaccount.com"
  state_bucket              = "${local.project_id}-iac-tfstate"

  common_labels = merge(
    {
      environment = var.environment
      managed_by  = "terraform"
    },
    var.labels,
  )
}
```

## variables.tf

```hcl
variable "environment" {
  description = "Deployment environment; everything else is derived from it. Pass via TF_VAR_environment (CI sets it from the branch)."
  type        = string

  validation {
    condition     = contains(["dev", "prod"], var.environment)
    error_message = "environment must be one of: dev, prod."
  }
}

variable "region" {
  description = "The default GCP region for regional resources."
  type        = string
  default     = "europe-west4"
}

variable "labels" {
  description = "Common labels applied to resources that support them."
  type        = map(string)
  default     = {}
}

variable "enabled_apis" {
  description = "GCP service APIs to enable for this project."
  type        = list(string)
  default = [
    "iamcredentials.googleapis.com", # Service account impersonation
  ]
}
```

## providers.tf

```hcl
terraform {
  required_version = "~> 1.12"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.12.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 7.12.0"
    }
  }
}

provider "google" {
  project                     = local.project_id
  region                      = var.region
  impersonate_service_account = local.terraform_service_account
}

provider "google-beta" {
  project                     = local.project_id
  region                      = var.region
  impersonate_service_account = local.terraform_service_account
}
```

## backend.tf (partial — only `prefix` is static)

```hcl
terraform {
  backend "gcs" {
    prefix = "terraform/state"
    # bucket + impersonate_service_account supplied at init via -backend-config
    # (derived from the environment — see README).
  }
}
```

## main.tf

```hcl
# Enable required GCP service APIs
resource "google_project_service" "this" {
  for_each = toset(var.enabled_apis)

  project            = local.project_id
  service            = each.value
  disable_on_destroy = false
}
```

Add real resources below. Apply `local.common_labels` to anything that supports labels. Use
`provider = google-beta` for resources/fields only available in beta.

## outputs.tf

Start empty. Add outputs with a `description` as resources are introduced, e.g.:

```hcl
output "project_id" {
  description = "The GCP project resources were deployed into."
  value       = local.project_id
}
```

## .gitignore (Loop's standard version)

```gitignore
# Terraform
**/.terraform/*
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
*.tfvars.json
!*.tfvars.example
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc
# Commit .terraform.lock.hcl (provider version reproducibility) — intentionally NOT ignored.

# Credentials / secrets
*.json
!*.tfvars.json
*.pem
*.key

# OS / editor
.DS_Store
.idea/
.vscode/
*.swp
```

## Local usage

Set the environment once, then pass the matching bucket + SA to `init`:

```bash
cd terraform
export TF_VAR_environment=dev

terraform init \
  -backend-config="bucket={{ ORG }}-{{ GROUP_ABBREV }}-${TF_VAR_environment}-{{ PROJECT_NAME }}-iac-tfstate" \
  -backend-config="impersonate_service_account=tf-iac-sa@{{ ORG }}-{{ GROUP_ABBREV }}-${TF_VAR_environment}-{{ PROJECT_NAME }}.iam.gserviceaccount.com"

terraform plan
terraform apply
```

Switching environment locally needs a backend re-init with the new bucket:

```bash
export TF_VAR_environment=prod
terraform init -reconfigure \
  -backend-config="bucket={{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }}-iac-tfstate" \
  -backend-config="impersonate_service_account=tf-iac-sa@{{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }}.iam.gserviceaccount.com"
terraform plan
```

> Keep `TF_VAR_environment` and the `-backend-config` bucket in the same environment — the
> `environment` validation and the derived `project_id` must match the state bucket you init.
> If `init` fails on a missing bucket/permissions, confirm `gcp-landingzone` provisioned that
> environment — never create the bucket from this repo.

## Reusable module template (moderate / complex tiers)

Each module under `terraform/modules/<name>/` has three files.

`modules/<name>/variables.tf`:

```hcl
variable "project_id" {
  description = "GCP project ID."
  type        = string
}

variable "labels" {
  description = "Labels to apply to resources that support them."
  type        = map(string)
  default     = {}
}
```

`modules/<name>/main.tf`: resources for the module (apply `var.labels` where supported).
`modules/<name>/outputs.tf`: values the root needs, each with a `description`.

Call from the root `terraform/main.tf`, passing the derived locals:

```hcl
module "<name>" {
  source     = "./modules/<name>"
  project_id = local.project_id
  labels     = local.common_labels
}
```
