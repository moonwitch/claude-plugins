# Repo documentation templates

Every scaffolded repo ships an `AGENTS.md` and a `README.md` at the root, using the
`{{ MUSTACHE }}` placeholders. Environments are branches (`dev`, `prod`); the only environment
input is the `environment` variable and `locals.tf` derives the rest. The GCP projects, buckets
and `tf-iac-sa` accounts come from `gcp-landingzone`.

> The `\``` fences below are escaped so the templates render inside this reference. When writing
> the actual files, use normal triple-backtick fences.

## README.md

```markdown
# {{ PROJECT_NAME }}

Infrastructure for the `{{ PROJECT_NAME }}` project, managed with Terraform. This project {{ PROJECT_DESCRIPTION }} and is provisioned on GCP using the `google` and `google-beta` providers.

## Environments are branches

One Terraform root module, one branch per environment. The `.tf` code is identical across
branches; the only environment input is the `environment` variable, and `locals.tf` derives the
project, service account and state bucket from it:

| Branch | GCP project | State bucket |
| --- | --- | --- |
| `dev`  | {{ ORG }}-{{ GROUP_ABBREV }}-dev-{{ PROJECT_NAME }}  | {{ ORG }}-{{ GROUP_ABBREV }}-dev-{{ PROJECT_NAME }}-iac-tfstate |
| `prod` | {{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }} | {{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }}-iac-tfstate |

## Layout

\```
.
├── terraform/
│   ├── providers.tf            # provider config & version constraints
│   ├── backend.tf              # partial GCS backend (bucket + SA via -backend-config)
│   ├── variables.tf            # environment, region, labels, enabled_apis
│   ├── locals.tf               # naming convention -> project_id, SA, bucket, labels
│   ├── main.tf                 # enabled APIs + shared resources
│   ├── <service>.tf            # one file per service (compute.tf, storage.tf, ...)
│   └── outputs.tf
├── AGENTS.md
├── README.md
└── .gitignore
\```

## Usage

The `environment` variable drives everything; pass it via `TF_VAR_environment` and hand the
matching bucket + SA to `init`:

\```sh
cd terraform
export TF_VAR_environment=dev

terraform init \
  -backend-config="bucket={{ ORG }}-{{ GROUP_ABBREV }}-${TF_VAR_environment}-{{ PROJECT_NAME }}-iac-tfstate" \
  -backend-config="impersonate_service_account=tf-iac-sa@{{ ORG }}-{{ GROUP_ABBREV }}-${TF_VAR_environment}-{{ PROJECT_NAME }}.iam.gserviceaccount.com"

terraform plan
terraform apply

# switch environment: re-init with the new bucket
export TF_VAR_environment=prod
terraform init -reconfigure \
  -backend-config="bucket={{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }}-iac-tfstate" \
  -backend-config="impersonate_service_account=tf-iac-sa@{{ ORG }}-{{ GROUP_ABBREV }}-prod-{{ PROJECT_NAME }}.iam.gserviceaccount.com"
\```

## Provisioning & remote state

The GCP projects, state buckets, and `tf-iac-sa` service accounts are provisioned by the
`gcp-landingzone` repo — this repo does not create them. State lives in each environment's GCS
bucket (prefix `terraform/state`). Terraform impersonates
`tf-iac-sa@<project-id>.iam.gserviceaccount.com`, so authenticate as a principal with
`roles/iam.serviceAccountTokenCreator` on it:

\```sh
gcloud auth application-default login
\```

## Conventions

• One file per service (`compute.tf`, `storage.tf`, …); shared/base resources in `main.tf`.
• Enable a service's API in `enabled_apis` (variables.tf) when you add its file.
• No service-account keys — impersonation replaces them. State files are git-ignored.
```

## AGENTS.md

```markdown
# AGENTS.md

Guidance for AI coding agents working in this repository.

## Overview

This repo manages infrastructure for the {{ PROJECT_NAME }} project using Terraform. Its purpose is to {{ PROJECT_DESCRIPTION }}. The root module lives in `terraform/` and targets GCP via the `hashicorp/google` and `hashicorp/google-beta` providers.

## Environments are branches

Each environment is a git branch (`dev`, `prod`). The `.tf` code is identical across branches.
The only environment input is the `environment` variable (set via `TF_VAR_environment`, which CI
takes from the branch); `locals.tf` derives `project_id`, the `tf-iac-sa` email and the state
bucket from it. Never hardcode env values. The GCP projects, buckets and SAs come from
`gcp-landingzone`; do not create them here.

## File-per-service convention

Each GCP service gets its own file: `compute.tf`, `storage.tf`, `vm.tf`, etc. Shared/base
resources (API enablement, project-wide IAM) live in `main.tf`. When the project grows complex,
promote a service's file into a module under `modules/<service>/` and call it from `main.tf`.
When you add a service, also add its API to `enabled_apis` in `variables.tf`.

## Setup & commands

All Terraform commands run from `terraform/`, with the environment exported and the bucket passed to init.

\```sh
cd terraform
export TF_VAR_environment=dev
terraform init -backend-config="bucket=...-iac-tfstate" -backend-config="impersonate_service_account=tf-iac-sa@...gserviceaccount.com"
terraform fmt
terraform validate
terraform plan
terraform apply            # requires review
\```

Switching environments needs `terraform init -reconfigure` with the new bucket.

## Conventions

- Run `terraform fmt` before committing; all `.tf` files must be formatted.
- Run `terraform validate` after changes and resolve errors.
- Declare new inputs in `variables.tf` with a `description` and `type`. Per-environment values
  are derived in `locals.tf` from `environment`, not hardcoded.
- Expose useful values in `outputs.tf` with a `description`.
- Apply `local.common_labels` to resources that support labels.
- Pin provider versions in `providers.tf` and the Terraform CLI version (`required_version`).
- Both `google` and `google-beta` are configured. Use `google-beta` (`provider = google-beta`)
  for resources/fields only available in beta.

## Authentication & impersonation

Terraform impersonates `tf-iac-sa@<project-id>.iam.gserviceaccount.com`, set on the providers
(`providers.tf`) and the backend (`-backend-config` at init). Authenticate as a principal with
`roles/iam.serviceAccountTokenCreator` on it (`gcloud auth application-default login`). Do not
commit service account keys.

## Safety & secrets

- Never commit secrets, credentials, or `*.tfvars`. State files and `.terraform/` are git-ignored.
- Do not run `terraform apply`/`destroy` or `-auto-approve` without explicit user confirmation.
- Treat `terraform plan` output as the source of truth; surface it before applying.

## Commits

- Keep commits focused with clear, imperative messages.
- `dev` and `prod` are long-lived environment branches. Do not create branches or push unless asked.
```
