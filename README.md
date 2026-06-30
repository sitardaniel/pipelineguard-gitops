# 🛡️ PipelineGuard - GitOps

> GitOps configuration repository for PipelineGuard. This is the single source of truth for cluster state - everything Argo CD deploys lives here.

[![GitOps: Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-orange)](https://argo-cd.readthedocs.io/)
[![Config: Helm](https://img.shields.io/badge/config-Helm-blue)](https://helm.sh/)
[![Policy: OPA](https://img.shields.io/badge/policy-OPA-purple)](https://www.openpolicyagent.org/)

---

## What This Repo Contains

This repo holds all Kubernetes manifests, Helm values, and Argo CD Application definitions for PipelineGuard. Nothing is applied to the cluster by hand - Argo CD watches this repo and reconciles the cluster to match.

| Repo | Purpose |
|------|---------|
| [`pipelineguard-app`](https://github.com/sitardaniel/pipelineguard-app) | Scanner source code + Dockerfiles |
| **`pipelineguard-gitops`** (this repo) | Kubernetes manifests + Argo CD apps |
| [`pipelineguard-infra`](https://github.com/sitardaniel/pipelineguard-infra) | Terraform/Terragrunt for AWS |

---

## Repository Structure

```
pipelineguard-gitops/
├── apps/
│   ├── pipelineguard-app.yaml      # Argo CD Application for scanner jobs
│   ├── grafana.yaml                # Argo CD Application for Grafana
│   ├── prometheus.yaml             # Argo CD Application for Prometheus
│   └── vault.yaml                  # Argo CD Application for Vault
├── manifests/
│   ├── namespaces/                 # Namespace definitions
│   ├── rbac/                       # Service accounts + ClusterRoles
│   ├── jobs/
│   │   ├── trivy-job.yaml
│   │   ├── checkov-job.yaml
│   │   ├── gitleaks-job.yaml
│   │   └── grype-job.yaml
│   ├── cronjobs/
│   │   └── scheduled-scan.yaml     # 24h scheduled full scan
│   └── webhook-receiver/           # Webhook receiver deployment
├── helm-values/
│   ├── grafana-values.yaml
│   ├── prometheus-values.yaml
│   └── vault-values.yaml
├── policies/
│   └── opa/                        # OPA Rego policy files
├── .github/
│   └── ISSUE_TEMPLATE/
├── SECURITY.md
└── README.md
```

---

## GitOps Principles

- **No manual `kubectl apply`** - all changes go through a PR to this repo
- **Argo CD auto-syncs** from `main` branch
- **Image tags are pinned** - no `latest` tags in production manifests
- **RBAC is explicit** - every component has a dedicated ServiceAccount with minimum permissions
- **Secrets are never stored here** - Vault handles all credentials; manifests only contain Vault path references

---

## Bootstrap (Local kind Cluster)

### Prerequisites

- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)
- [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

### Steps

```bash
# 1. Create kind cluster
kind create cluster --name pipelineguard --config kind-config.yaml

# 2. Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for Argo CD to be ready
kubectl wait --for=condition=available --timeout=120s deployment/argocd-server -n argocd

# 4. Apply the root app (app-of-apps pattern)
kubectl apply -f apps/ -n argocd

# 5. Port-forward Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Argo CD will now watch this repo and deploy all components automatically.

---

## Environments

| Environment | Cluster        | Branch |
|-------------|----------------|--------|
| Local dev   | kind           | `main` |
| Demo (AWS)  | EKS (on-demand)| `main` |

The EKS cluster is spun up only for demos using `pipelineguard-infra`, then torn down to minimize cost (~$5–15 per demo session).

---

## Security

See [SECURITY.md](SECURITY.md).
