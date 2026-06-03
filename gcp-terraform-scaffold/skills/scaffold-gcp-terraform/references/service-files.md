# File-per-service convention

Loop organizes resources by service, one file per service: `compute.tf`, `vm.tf`,
`storage.tf`, `network.tf`, and so on. Shared/base resources (API enablement, project-wide
IAM) stay in `main.tf`.

- **Simple / moderate:** each service is a `.tf` file at the `terraform/` root.
- **Complex:** promote each service file into a module — `terraform/modules/<service>/` — and
  call it from `main.tf` with the derived locals.

Always ask the user **which services to include** and **which APIs to enable** before
generating. Create one file (or module) per chosen service, and add each service's API(s) to
`enabled_apis` in `variables.tf`.

## Service → API map (common GCP services)

Use this to populate `enabled_apis` and to name service files. Confirm exact APIs with the user;
this is a starting set, not exhaustive.

| Service | Suggested file | API(s) to enable |
| --- | --- | --- |
| Compute Engine / VMs | `compute.tf` (or `vm.tf`) | `compute.googleapis.com` |
| Cloud Storage (GCS) | `storage.tf` | `storage.googleapis.com` |
| Cloud Run | `run.tf` | `run.googleapis.com`, `artifactregistry.googleapis.com` |
| GKE | `gke.tf` | `container.googleapis.com` |
| Cloud SQL | `sql.tf` | `sqladmin.googleapis.com` |
| Pub/Sub | `pubsub.tf` | `pubsub.googleapis.com` |
| BigQuery | `bigquery.tf` | `bigquery.googleapis.com` |
| Networking (VPC) | `network.tf` | `compute.googleapis.com` |
| Cloud Functions | `functions.tf` | `cloudfunctions.googleapis.com`, `cloudbuild.googleapis.com` |
| Secret Manager | `secrets.tf` | `secretmanager.googleapis.com` |
| Artifact Registry | `artifacts.tf` | `artifactregistry.googleapis.com` |
| IAM / service accounts | `iam.tf` | `iam.googleapis.com`, `iamcredentials.googleapis.com` |

`iamcredentials.googleapis.com` is always enabled (impersonation). Merge the chosen services'
APIs into `enabled_apis`, de-duplicated.

## Per-service file stub (simple / moderate — root file)

`compute.tf` (example):

```hcl
# Compute Engine resources for {{ PROJECT_NAME }}.
# API enabled via var.enabled_apis -> google_project_service in main.tf.

# resource "google_compute_instance" "example" {
#   name         = "${local.project_id}-example"
#   machine_type = "e2-small"
#   zone         = "${var.region}-b"
#   project      = local.project_id
#   labels       = local.common_labels
#   # ...
# }
```

Keep each file focused on one service. Reference `local.project_id`, `var.region`, and
`local.common_labels`. Add real resources or leave a clearly commented stub for the user.

## Per-service module (complex — modules/<service>/)

`modules/compute/variables.tf`:

```hcl
variable "project_id" {
  description = "GCP project ID."
  type        = string
}

variable "region" {
  description = "Default region."
  type        = string
}

variable "labels" {
  description = "Labels to apply."
  type        = map(string)
  default     = {}
}
```

`modules/compute/main.tf`: the service's resources (use `var.project_id`, `var.region`, `var.labels`).
`modules/compute/outputs.tf`: values the root needs, each with a `description`.

Call it from the root `terraform/main.tf`:

```hcl
module "compute" {
  source     = "./modules/compute"
  project_id = local.project_id
  region     = var.region
  labels     = local.common_labels
}
```

## Updating enabled_apis

When services are chosen, set the default in `variables.tf`, e.g. for Compute + Storage:

```hcl
variable "enabled_apis" {
  description = "GCP service APIs to enable for this project."
  type        = list(string)
  default = [
    "iamcredentials.googleapis.com", # Service account impersonation (always)
    "compute.googleapis.com",        # Compute Engine
    "storage.googleapis.com",        # Cloud Storage
  ]
}
```
