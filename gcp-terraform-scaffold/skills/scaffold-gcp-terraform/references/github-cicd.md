# GitHub Actions CI/CD for Terraform

Create `.github/workflows/terraform.yml`. Environments are branches (`dev`, `prod`). The
workflow derives the environment from the branch, exports it as `TF_VAR_environment` (so the
`locals` resolve), and computes the matching state bucket + impersonation SA for
`terraform init -backend-config`. It runs `fmt`/`init`/`validate`/`plan` on pull requests and
`apply` on push.

## Single source of truth

The branch name **is** the environment. From it the workflow sets `TF_VAR_environment` and
computes `bucket` / `impersonate_service_account` using the naming convention
`<org>-<group>-<env>-<project>`. Nothing env-specific is committed in the repo.

## How auth fits Loop's impersonation model

CI can't run `gcloud auth application-default login`, so it authenticates via **Workload
Identity Federation** for a short-lived identity, and Terraform still **impersonates `tf-iac-sa`**
through the provider config and the backend's `impersonate_service_account`. Requirement: the
WIF-bound CI service account holds `roles/iam.serviceAccountTokenCreator` on the per-env
`tf-iac-sa`. Keep WIF values as secrets.

## Workflow

Substitute `{{ ORG }}`, `{{ GROUP_ABBREV }}`, `{{ PROJECT_NAME }}` when scaffolding.

```yaml
name: terraform

on:
  pull_request:
    branches: [dev, prod]
  push:
    branches: [dev, prod]

permissions:
  contents: read
  id-token: write          # required for Workload Identity Federation
  pull-requests: write

env:
  TF_VERSION: "1.12.0"
  ORG: "{{ ORG }}"
  GROUP: "{{ GROUP_ABBREV }}"
  PROJECT: "{{ PROJECT_NAME }}"

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform
    steps:
      - uses: actions/checkout@v4

      - name: Derive environment + names from branch
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            ENV="${{ github.base_ref }}"
          else
            ENV="${{ github.ref_name }}"
          fi
          PROJECT_ID="${ORG}-${GROUP}-${ENV}-${PROJECT}"
          {
            echo "TF_VAR_environment=${ENV}"
            echo "STATE_BUCKET=${PROJECT_ID}-iac-tfstate"
            echo "TF_SA=tf-iac-sa@${PROJECT_ID}.iam.gserviceaccount.com"
          } >> "$GITHUB_ENV"

      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_CI_SA }}   # has tokenCreator on tf-iac-sa

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - run: terraform fmt -check -recursive

      - name: Init
        run: |
          terraform init \
            -backend-config="bucket=${STATE_BUCKET}" \
            -backend-config="impersonate_service_account=${TF_SA}"

      - run: terraform validate

      - name: Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -input=false

      - name: Apply
        if: github.event_name == 'push'
        run: terraform apply -auto-approve -input=false
```

`TF_VAR_environment` flows into the `locals`, so `project_id`, the SA, and labels all resolve
automatically — `plan`/`apply` need no extra `-var`.

Protect the `prod` branch with required reviewers so production changes are gated before merge.

## Required GitHub secrets (document in the repo README)

| Secret | Purpose |
| --- | --- |
| `GCP_WIF_PROVIDER` | Workload Identity provider resource name (`projects/.../providers/...`). |
| `GCP_CI_SA` | CI service account email, granted `roles/iam.serviceAccountTokenCreator` on each env's `tf-iac-sa`. |

## Setup note for the repo README

Workload Identity Federation is configured once in GCP (pool + provider + a CI service account
bound to the GitHub repo, with `tokenCreator` on each `tf-iac-sa`). Until the secrets are set,
the auth step fails — expected on a fresh scaffold. The state buckets come from `gcp-landingzone`.
