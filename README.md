# Restore Cluster and Deploy ArgoCD (Matrix + Aggregation)

This workflow restores an existing cluster configuration (optionally from another source cluster), commits any restored configuration to GitHub, and redeploys all ArgoCD-managed applications using matrix-based parallelism.  
It then aggregates the deployment results into a single combined summary.

---

## 🧩 Overview

This workflow is part of the GitOps automation framework. It coordinates full cluster restoration and application re-deployment by leveraging reusable composite actions.  
It performs the following high-level steps:

1. **Restore cluster** configuration (copy or reuse existing directory).  
2. **Commit and merge** restored configuration changes via Auto-Commit-Squash-Merge.  
3. **Generate a deployment matrix** dynamically from `create.json` files.  
4. **Deploy ArgoCD applications** for each namespace/app in parallel using the ArgoCD-Manage-Applications action.  
5. **Aggregate results** from all parallel jobs into a unified summary report.

---

## 🌐 GitOps Manager™ Enterprise Platform

[**GitOps Manager™ Enterprise**](https://gitopsmanager.io) is the full platform that powers this open-source workflow.  
It’s a **turnkey GitOps automation platform** for AWS and Azure — combining open-source GitHub Actions, Kubernetes infrastructure automation, and global-scale CI/CD.

**Highlights:**
- Secure, opinionated **multi-cloud GitOps automation** for Kubernetes workloads.  
- Deep integration with **ArgoCD**, **Argo Workflows**, **Traefik**, **ECK**, and **Kubernetes Dashboard**.  
- Built for **high availability**, **autoscaling**, and **managed upgrades**.  
- Supports **Workload Identity**, **Pod Identity**, and **private, network-isolated clusters**.  
- Enables **global deployments**, **secret management**, and **production-grade infrastructure** with **zero vendor lock-in**.

🔗 Learn more: [https://gitopsmanager.io](https://gitopsmanager.io)

---

## 🧩 Dependencies

| Action | Description | Repository |
|--------|--------------|-------------|
| **Auto-Commit-Squash-Merge** | Commits and merges restored cluster configuration | [`gitopsmanager/Auto-Commit-Squash-Merge`](https://github.com/gitopsmanager/Auto-Commit-Squash-Merge) |
| **ArgoCD-Manage-Applications** | Handles ArgoCD app lifecycle (create, sync, delete, wait-for-sync) | [`gitopsmanager/ArgoCD-Manage-Applications`](https://github.com/gitopsmanager/ArgoCD-Manage-Applications) |
| **create-github-app-token** | Generates short-lived GitHub App tokens | [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) |
| **github-script** | Executes inline JavaScript for logic, JSON parsing, and summaries | [`actions/github-script`](https://github.com/actions/github-script) |
| **checkout** | Clones the target repository for modification | [`actions/checkout`](https://github.com/actions/checkout) |
| **upload-artifact / download-artifact** | Stores and retrieves summary artifacts between jobs | [`actions/upload-artifact`](https://github.com/actions/upload-artifact), [`actions/download-artifact`](https://github.com/actions/download-artifact) |

---

## 🚀 Workflow Inputs

| Name | Description | Required | Default |
|------|--------------|-----------|----------|
| `ClusterRestoreSource` | Optional: source cluster directory name | ❌ No | `""` |
| `ClusterRestoreTarget` | Target cluster directory name | ✅ Yes | — |
| `namespace_filter` | Optional comma-separated list of namespaces or namespace:app pairs | ❌ No | `""` |
| `cd_repo_org` | GitHub org/owner of the continuous-deployment repo | ✅ Yes | — |
| `cd_repo` | Continuous-deployment repository name | ✅ Yes | — |
| `github_runner` | **Self-hosted runner label** used for deployment and restore jobs (e.g. `self-hosted,deployer`) | ✅ Yes | — |
| `insecure_argo` | Skip SSL verification for ArgoCD | ❌ No | `false` |
| `delete_first` | Delete applications before redeploying | ❌ No | `false` |
| `delete_only` | Delete applications without redeploying | ❌ No | `false` |
| `skip_status_check` | Skip waiting for ArgoCD sync status | ❌ No | `false` |

---

## 🔒 Secrets

This workflow requires several GitHub **secrets** for authentication and secure operations.  
Secrets are never logged and must be defined in the **calling repository** (or organization) before triggering the workflow.

| Secret | Required | Description |
|---------|-----------|-------------|
| **`ARGOCD_AUTH_TOKEN`** | ❌ | ArgoCD API authentication token. **Preferred method** for automation. Provide **this OR** the username/password pair below. |
| **`ARGOCD_USERNAME`** | ❌ | ArgoCD username (used if `ARGOCD_AUTH_TOKEN` is not set). Must be provided **together with** `ARGOCD_PASSWORD`. |
| **`ARGOCD_PASSWORD`** | ❌ | ArgoCD password (used if `ARGOCD_AUTH_TOKEN` is not set). Must be provided **together with** `ARGOCD_USERNAME`. |
| **`ARGOCD_CA_CERT`** | ❌ | Optional custom CA certificate for ArgoCD. Use when ArgoCD uses a self-signed or private certificate authority. |
| **`CONTINUOUS_DEPLOYMENT_GH_APP_ID`** | ✅ | GitHub App ID that has **write access** to the continuous-deployment repository. Used to generate a short-lived token for secure commits. |
| **`CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY`** | ✅ | Private key for the GitHub App above. Required to sign and authenticate token requests. |

### 🔐 Notes

- You **must provide either** `ARGOCD_AUTH_TOKEN` **or** the combination of `ARGOCD_USERNAME` and `ARGOCD_PASSWORD`.  
- The GitHub App secrets (`CONTINUOUS_DEPLOYMENT_GH_APP_ID` and `CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY`) are **always required**.  
- Store secrets under **Repository Settings → Secrets and Variables → Actions** before running the workflow.

---

## ⚙️ Job Summary

### 🧱 `restore`
- Runs on the runner specified by the `github_runner` input (e.g., `self-hosted,deployer`).  
- Validates input directories and performs optional copy from source cluster.  
- If a new cluster directory is created, commits it via Auto-Commit-Squash-Merge.  
- Builds a deployment matrix from all `create.json` files.

### 🚀 `deploy`
- Executes parallel deployments (matrix) on `[self-hosted, deployer]` or another runner as configured by `github_runner`.  
- Deploys applications in each namespace using ArgoCD-Manage-Applications.  
- Uploads a per-namespace deployment summary.

### 📊 `aggregate`
- Downloads all deployment summaries.  
- Combines them into a single `combined-summary.json` file.  
- Adds a Markdown summary table to the workflow summary view.

---

## 🧠 Example Call

```yaml
jobs:
  restore:
    uses: gitopsmanager/k8s-deploy/.github/workflows/k8s-restore-cluster.yml@v2
    with:
      ClusterRestoreSource: "cluster-staging"
      ClusterRestoreTarget: "cluster-prod"
      cd_repo_org: "affinity7software"
      cd_repo: "continuous-deployment"
      github_runner: "self-hosted,deployer"
      delete_first: false
      delete_only: false
    secrets:
      ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
      CONTINUOUS_DEPLOYMENT_GH_APP_ID: ${{ secrets.CONTINUOUS_DEPLOYMENT_GH_APP_ID }}
      CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY: ${{ secrets.CONTINUOUS_DEPLOYMENT_GH_APP_PRIVATE_KEY }}


---

## 📄 License

MIT License © Affinity7 Consulting Ltd
