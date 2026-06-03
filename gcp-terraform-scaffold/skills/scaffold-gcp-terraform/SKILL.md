---
name: scaffold-gcp-terraform
description: Scaffolds a new Terraform project for Google Cloud (GCP) in Loop's house style, sized to project complexity. Use when someone wants to start a new GCP infrastructure repo, bootstrap Terraform for Google Cloud, scaffold a terraform project, set up IaC for GCP, create a new GCP project structure, or set up a project Terraform repo for the platform team. Produces one repo per project with a single terraform root module, environments as git branches (dev and prod) driven by a single environment variable with everything else derived in locals, a partial GCS backend, a file-per-service layout (compute.tf, storage.tf, and so on, promoted to modules when complex), google and google-beta provider pins, service-account impersonation, AGENTS and README docs, a branch-aware GitHub Actions workflow, and optionally creates and pushes the GitHub repo with branches via the gh CLI. Assumes the gcp-landingzone repo already provisioned the projects, buckets, and service accounts.
---

# Scaffold GCP Terraform

Generate a Terraform project for Google Cloud using Loop's house conventions, sized to the
complexity of the work.

## Core model — read first

- **One repo per project.** A single Terraform root module lives in `terraform/`.
- **Environments are git branches** (`dev`, `prod`). The `.tf` code is identical across branches.
- **`environment` is the only environment input.** `locals.tf` derives `project_id`, the
  `tf-iac-sa` email, and the state bucket from it via the naming convention. Pass it as
  `TF_VAR_environment` (CI sets it from the branch).
- **Backend bucket is the one external value.** A `backend` block can't read variables, so the
  bucket + impersonation SA are passed to `terraform init` via `-backend-config` (computed from
  `environment`); only the static `prefix` lives in the block.
- **File per service.** Each GCP service gets its own file (`compute.tf`, `storage.tf`, …);
  shared/base resources stay in `main.tf`. When complex, services become `modules/<service>/`.
- **`gcp-landingzone` owns base infra.** Projects, buckets, and `tf-iac-sa` accounts already
  exist. Consume them — never create projects/buckets/SAs, never add a `bootstrap/` layer.
- **Naming:** `{{ ORG }}-{{ GROUP_ABBREV }}-<env>-{{ PROJECT_NAME }}`; bucket `<project-id>-iac-tfstate`,
  prefix `terraform/state`; SA `tf-iac-sa@<project-id>.iam.gserviceaccount.com`.
- **Providers:** `google` + `google-beta`, version-pinned, `required_version` in `providers.tf`.

## Workflow

Follow in order. Confirm inputs, services, and complexity before writing files.

### 1. Gather inputs

Collect (ask only for what is missing). They fill the `{{ MUSTACHE }}` placeholders:

- **`{{ ORG }}`** — org short name. Default `loop`.
- **`{{ GROUP_ABBREV }}`** — group/domain abbrev (e.g. `pla` for platform).
- **`{{ PROJECT_NAME }}`** — project short name (kebab-case); names the repo and project ID.
- **`{{ PROJECT_DESCRIPTION }}`** — one line on what the project does.
- **Environments** — default `dev` + `prod`; one branch per environment.
- **Region** — default `europe-west4`.

Confirm the composed per-env project IDs back to the user.

### 2. Ask which services / APIs to enable

**Always ask** which GCP services this project will use (e.g. Compute, Cloud Storage, Cloud Run,
GKE, Cloud SQL, Pub/Sub, networking, IAM). Read `references/service-files.md` for the
service → API map. From the answer:

- Create one file per service (`<service>.tf`) — or one module per service when complex.
- Populate `enabled_apis` in `variables.tf` with the union of those services' APIs (always
  including `iamcredentials.googleapis.com`).

If the user is unsure, scaffold with no service files yet (just the baseline) and note they can
add `<service>.tf` files later.

### 3. Assess complexity

Read `references/complexity-tiers.md` and classify as **simple**, **moderate**, or **complex**.
Environments are NOT a complexity axis (always branches) — complexity is module structure:

- **Simple** — service files at the `terraform/` root (default).
- **Moderate** — a few shared modules under `terraform/modules/` plus root service files.
- **Complex** — each service promoted to `modules/<service>/`, called from `main.tf`.

State your assessment in a sentence; let the user override. When unsure, choose **simple**.

### 4. Generate the structure

Read `references/terraform-templates.md` (core files) and `references/service-files.md`
(per-service files/modules). Substitute every `{{ MUSTACHE }}` placeholder. Always include:

- `terraform/providers.tf` — version + `google`/`google-beta` pins + provider blocks using `local.project_id` / `local.terraform_service_account`.
- `terraform/backend.tf` — partial: `backend "gcs" { prefix = "terraform/state" }`.
- `terraform/variables.tf` — `environment` (required, validated), `region`, `labels`, `enabled_apis` (populated from the chosen services).
- `terraform/locals.tf` — naming constants + derived `project_id` / SA / bucket + `common_labels`.
- `terraform/main.tf` — `google_project_service` over `enabled_apis` + any shared resources + module calls (complex).
- One `<service>.tf` per chosen service (or `modules/<service>/` when complex).
- `terraform/outputs.tf` — starts empty.
- `.gitignore` — Loop's standard version.

Never create a local `terraform.tfvars`, never add bucket/project/SA creation.

### 5. Generate repo docs

Read `references/repo-docs-templates.md` and create `AGENTS.md` and `README.md` at the repo
root, substituting placeholders (including the per-env branch/project/bucket table).

### 6. Add CI/CD

Read `references/github-cicd.md` and create `.github/workflows/terraform.yml`. It derives the
environment from the branch, exports `TF_VAR_environment`, computes the bucket + SA for
`init -backend-config`, runs `fmt`/`init`/`validate`/`plan` on PRs and `apply` on push, and
authenticates via Workload Identity Federation then impersonates `tf-iac-sa`. Leave WIF values
as clearly marked secret placeholders.

### 7. Create the GitHub repo + branches (if requested)

Use the `gh` CLI via Bash. Check `gh auth status` first; if not authenticated, stop and tell the
user to run `gh auth login`. Then from the repo root:

```bash
git init -b prod
git add -A
git commit -m "chore: scaffold GCP Terraform project (<tier>)"
gh repo create <org-or-user>/<project-name> --private --source=. --remote=origin --push
git branch dev && git push -u origin dev
```

Confirm owner/org, private-vs-public (default private), and default/protected branch (default
`prod`) before running. Create one branch per environment. Never force-push or touch an existing
repo without explicit confirmation.

### 8. Summarize

Report the tier, the services/files created, the branches, and remaining `TODO`s (WIF secrets;
confirm `gcp-landingzone` provisioned each env's project/bucket/SA). Give the next commands:
`export TF_VAR_environment=dev`, then `terraform init -backend-config=...` and `terraform plan`.
Remind them to protect the `prod` branch.

## Guardrails

- Never create projects, buckets, or service accounts — `gcp-landingzone` owns them.
- Never write real credentials or SA keys — impersonation replaces them.
- Keep env values out of the `.tf` files; derive them in `locals.tf` from `environment`.
- One file per service; promote to `modules/<service>/` when complex.
- Pin provider and Terraform versions — never floating.
- Remote state from day one (partial backend); never local-only state.
- Never run `terraform apply`/`destroy` or `-auto-approve` outside the gated CI step without explicit user confirmation.
- This plugin targets Loop's internal team — write concrete references (gh CLI, GCS, impersonation, landing zone).
