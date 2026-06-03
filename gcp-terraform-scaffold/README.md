# gcp-terraform-scaffold

A Cowork plugin that scaffolds a new Terraform project for Google Cloud in Loop's house style: **one repo per project, environments as git branches, one file per service**.

## What it does

Ask Claude to start a new GCP Terraform project. The plugin's skill will:

1. **Gather naming inputs** — org, group abbrev, project name, description — composing per-environment project IDs `<org>-<group-abbrev>-<env>-<project>` (e.g. `loop-pla-dev-mcp`).
2. **Ask which services/APIs** the project needs (Compute, Storage, Cloud Run, GKE, …) and create one `.tf` file per service, enabling the matching APIs.
3. **Assess complexity** — *simple* (service files at the root), *moderate*/*complex* (services promoted to `modules/`). You can override.
4. **Generate the structure** — a single `terraform/` root module driven by one `environment` variable, with everything (project ID, SA, bucket) derived in `locals.tf`, a partial GCS backend, `google` + `google-beta` providers pinned and impersonation-configured.
5. **Write repo docs** — `AGENTS.md` and `README.md`.
6. **Add CI/CD** — a branch-aware GitHub Actions workflow that derives the environment from the branch, computes the bucket for `init`, runs `fmt`/`init`/`validate`/`plan` on PRs and `apply` on push, authenticating via Workload Identity Federation then impersonating `tf-iac-sa`.
7. **Create the GitHub repo + branches** (optional) — via `gh`, with `dev`/`prod` branches.

## The model

- **One repo per project.** Single Terraform root module under `terraform/`.
- **Environments are branches** (`dev`, `prod`). The only environment input is the `environment` variable; `locals.tf` derives the project ID, `tf-iac-sa`, and state bucket from it. The bucket is passed to `terraform init` via `-backend-config` (the one value a backend block can't compute itself).
- **File per service** — `compute.tf`, `storage.tf`, etc.; shared/base resources in `main.tf`. Promoted to `modules/<service>/` when complex.
- **`gcp-landingzone` owns base infra.** Projects, buckets, and `tf-iac-sa` accounts already exist — this scaffold consumes them.

## How to use it

> "Scaffold a new GCP Terraform project — loop, platform group, for a billing exporter using Compute and Cloud Storage, dev and prod. Create the repo too."

Claude composes the project IDs, asks which services, assesses the tier, generates the files, and (if asked) creates the repo with branches.

## Conventions baked in

- Project ID `<org>-<group-abbrev>-<env>-<project>`; bucket `<project-id>-iac-tfstate`, prefix `terraform/state`.
- One `environment` input; project/SA/bucket derived in `locals.tf`.
- Service-account impersonation (`tf-iac-sa@<project-id>...`) on both providers and the backend — no key files.
- `required_version` in `providers.tf` (no separate `versions.tf`).
- One file per service; `enabled_apis` lists the services' APIs.

## Prerequisites

- **`gcp-landingzone`** must have provisioned each environment's project, state bucket, and `tf-iac-sa` before `terraform init`.
- **`gh` CLI** authenticated (`gh auth login`) — only if you want the repo created automatically.
- To run Terraform: `export TF_VAR_environment=<env>`, then `gcloud auth application-default login` as a principal with `roles/iam.serviceAccountTokenCreator` on `tf-iac-sa`.
- For CI/CD: GitHub secrets `GCP_WIF_PROVIDER` and `GCP_CI_SA` once Workload Identity Federation is configured.

## What's inside

- `skills/scaffold-gcp-terraform/` — the skill, with references for complexity tiers, Terraform templates, the file-per-service convention + service→API map, repo docs (README + AGENTS.md), and the GitHub Actions workflow.

No custom MCP server is bundled — repo creation uses the `gh` CLI directly.
