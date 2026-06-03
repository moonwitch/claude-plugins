# Complexity tiers

Every scaffold is **one repo per project** with a single Terraform root module under
`terraform/`. **Environments are git branches** (`dev`, `prod`); the only environment input is
the `environment` variable, and `locals.tf` derives the project ID, `tf-iac-sa`, and state
bucket from it. The state bucket is passed to `terraform init` via `-backend-config` (computed
from `environment`) because a backend block can't read variables. The `gcp-landingzone` repo
owns the projects, buckets, and service accounts — this scaffold consumes them.

Complexity is **only** about how much module structure the root needs — not environments. When
unsure, choose **simple** and grow into modules later.

## Simple

**Use when:** a focused project with a handful of resources. The default.

```
<repo>/
├── terraform/
│   ├── providers.tf
│   ├── backend.tf            # partial: prefix static; bucket + SA via -backend-config
│   ├── variables.tf          # environment (required), region, labels, enabled_apis
│   ├── locals.tf             # naming -> project_id, tf SA, bucket, common_labels
│   ├── main.tf               # google_project_service + resources
│   └── outputs.tf
├── .github/workflows/terraform.yml
├── AGENTS.md
├── README.md
└── .gitignore
```

## Moderate

**Use when:** several distinct concerns worth isolating into reusable modules (e.g. iam,
networking, a service). Same single root + branch envs — add a module library the root calls
with the derived locals.

```
<repo>/
├── terraform/
│   ├── providers.tf  backend.tf  variables.tf  locals.tf  main.tf  outputs.tf
│   └── modules/
│       ├── iam/            # main.tf, variables.tf, outputs.tf
│       └── <other>/
├── .github/workflows/terraform.yml
├── AGENTS.md  README.md  .gitignore
```

The root `main.tf` wires modules with `module "<name>" { source = "./modules/<name>"; project_id = local.project_id; ... }`.

## Complex

**Use when:** many resources across several domains and a richer shared-module library.
Structurally the same as moderate (single root, branch envs, derived locals) with more modules.
Still no `bootstrap/` layer — landing zone owns base infra.

```
<repo>/
├── terraform/
│   ├── providers.tf  backend.tf  variables.tf  locals.tf  main.tf  outputs.tf
│   └── modules/
│       ├── iam/  networking/  <domain-modules>/
├── .github/workflows/terraform.yml
├── AGENTS.md  README.md  .gitignore
```

For complex, fully build one or two modules and leave clearly marked stubs for the rest so the
user copies the pattern rather than hand-building each one.
