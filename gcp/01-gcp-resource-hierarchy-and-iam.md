# Google Cloud Platform (GCP) Resource Hierarchy & IAM Architecture

GCP organizes resources hierarchically to enable centralized governance, policy inheritance, and granular access control across an enterprise.

---

## 1. Enterprise Resource Hierarchy

```text
Organization (e.g., yahyaoncloud.com)
│   ├── Org Policies (e.g. restrict public IP, enforce domain restriction)
│   └── Top-Level Folders
│
├── Production Folder
│   ├── Networking Project (Shared VPC Host)
│   ├── Workload-Core Project (GKE, Cloud Run)
│   └── Data Project (BigQuery, Cloud Spanner)
│
├── Non-Production / Staging Folder
│   ├── Staging-Network Project
│   └── Staging-Workloads Project
│
└── Shared Services Folder
    ├── CI/CD & Artifact Registry Project
    └── Security & Audit Logging Project (BigQuery / Cloud Storage Sink)
```

### Hierarchy Inheritance Rules
1. **Downwards Inheritance**: Permissions and Organization Policies applied at higher nodes (Org or Folder) are inherited by all child resources.
2. **Union of Permissions**: Effective permissions on a resource are the union of permissions granted at that resource plus all inherited parent permissions. You cannot restrict an inherited permission lower down using standard IAM (use Organization Policies or Deny Policies instead).
3. **Deny Policies**: IAM Deny policies take precedence over allow policies and block specific actions regardless of where allow is granted.

---

## 2. Cloud IAM Architecture

| Component | Definition | Production Best Practice |
| :--- | :--- | :--- |
| **Identities (Principals)** | Google Account, Google Workspace / Cloud Identity Group, Service Account. | Always bind roles to **Groups**, never individual user accounts. |
| **Roles** | Collections of granular permissions (`service.resource.verb`). | Use **Predefined Roles** (e.g., `roles/compute.networkAdmin`) or custom roles; avoid primitive (`Owner`, `Editor`, `Viewer`). |
| **Service Accounts (SA)** | Special machine accounts used by VMs, containers, and applications to call GCP APIs. | Disable automatic service account key downloads; use **Workload Identity** or **Attached Service Accounts**. |
| **Workload Identity Federation** | Authenticates external workloads (AWS, GitHub Actions, on-prem) into GCP without long-lived keys. | Use OIDC tokens exchanged via Google STS for temporary short-lived GCP access. |

---

## 3. GitHub Actions to GCP via Workload Identity Federation (Zero-Key Setup)

```yaml
# .github/workflows/deploy-gcp.yml
name: Deploy to GCP
on:
  push:
    branches: [ main ]

jobs:
  auth-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # Required for requesting the JWT
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-deployer@yahyaoncloud-prod.iam.gserviceaccount.com'

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Verify GCP Authentication
        run: gcloud compute instances list
```

---

## 4. Key CLI Commands

```bash
# List all active projects in Organization
gcloud projects list

# Describe effective IAM policy on a project
gcloud projects get-iam-policy my-project-id

# Grant a role to a group or service account
gcloud projects add-iam-policy-binding my-project-id \
  --member="group:cloud-engineers@yahyaoncloud.com" \
  --role="roles/viewer"

# Create a dedicated least-privilege service account
gcloud iam service-accounts create gke-node-sa \
  --display-name="GKE Worker Nodes SA"
```
