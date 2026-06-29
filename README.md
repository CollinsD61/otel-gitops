# otel-gitops

> **Production-style GitOps repository** for deploying the
> [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)
> to Kubernetes using **ArgoCD** and **Helm**.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Quick Start (Kind)](#quick-start-kind)
- [Environments](#environments)
- [GitOps Workflow](#gitops-workflow)
- [Extending the Repository](#extending-the-repository)
- [Documentation](#documentation)

---

## Overview

This repository follows the **GitOps** philosophy — the cluster state is
declared entirely in Git.  ArgoCD continuously reconciles the live cluster
state with what is defined here.

Key design decisions:

| Decision | Choice |
|----------|--------|
| GitOps engine | ArgoCD (App of Apps pattern) |
| Package manager | Helm (upstream chart, not vendored) |
| Local cluster | Kind |
| Environments | `dev` · `staging` · `prod` |
| Ingress | NGINX Ingress Controller |
| TLS | cert-manager (Let's Encrypt) |
| Metrics | metrics-server |

---

## Architecture

```
Developer
   |  git push
   v
GitHub (otel-gitops)
   |  webhook / poll
   v
ArgoCD (bootstrap/root-app.yaml)
   |  App of Apps
   v
ArgoCD Applications (applications/<env>/otel-demo.yaml)
   |  Helm render
   v
Kubernetes Cluster (Kind / EKS / GKE / AKS)
   |
   +-- otel-demo namespace        (OpenTelemetry Demo)
   +-- ingress-nginx namespace    (NGINX Ingress)
   +-- cert-manager namespace     (TLS)
   +-- monitoring namespace       (Prometheus / Grafana -- TODO)
```

See `docs/architecture.md` for a detailed walkthrough.

---

## Repository Structure

```
otel-gitops/
+-- bootstrap/              # ArgoCD App of Apps root -- deploy this first
|   +-- root-app.yaml
|   +-- argocd/
|   +-- ingress-nginx/
|   +-- metrics-server/
|   +-- cert-manager/
+-- applications/           # One ArgoCD Application per environment
|   +-- dev/
|   +-- staging/
|   +-- prod/
+-- environments/           # Helm values per environment
|   +-- dev/
|   +-- staging/
|   +-- prod/
+-- manifests/              # Raw Kubernetes manifests (namespaces, HPA, ...)
|   +-- namespaces/
|   +-- networkpolicy/
|   +-- hpa/
|   +-- ingress/
+-- docs/                   # Architecture & operational runbook
    +-- architecture.md
    +-- runbook.md
```

---

## Prerequisites

| Tool | Minimum version | Install |
|------|----------------|---------|
| kubectl | 1.28+ | https://kubernetes.io/docs/tasks/tools/ |
| helm | 3.14+ | https://helm.sh/docs/intro/install/ |
| argocd CLI | 2.10+ | https://argo-cd.readthedocs.io/en/stable/cli_installation/ |
| kind | 0.22+ | https://kind.sigs.k8s.io/docs/user/quick-start/ |
| docker | 24+ | https://docs.docker.com/get-docker/ |

---

## Quick Start (Kind)

```bash
# 1. Create a local Kind cluster
kind create cluster --name otel-gitops --config bootstrap/kind-config.yaml

# 2. Install ArgoCD into the cluster
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for ArgoCD to be ready
kubectl rollout status deploy/argocd-server -n argocd

# 4. Apply the root App of Apps manifest
kubectl apply -f bootstrap/root-app.yaml

# 5. Open the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Navigate to https://localhost:8080

# 6. Retrieve the initial admin password
argocd admin initial-password -n argocd
```

---

## Environments

| Environment | Namespace | Sync | Replicas | Resources |
|-------------|-----------|------|----------|-----------|
| dev | otel-demo | Automated | Low | Minimal |
| staging | otel-demo | Automated | Medium | Moderate |
| prod | otel-demo | Automated + selfHeal | High | Full |

Environment-specific Helm values live in `environments/<env>/values.yaml`.

---

## GitOps Workflow

1. **Branch** from `main` and make your changes (e.g., bump image tag).
2. Open a **Pull Request** -- CI validates YAML / Helm lint.
3. **Merge** to `main`.
4. ArgoCD detects drift and **auto-syncs** within 3 minutes (or immediately via webhook).
5. Verify rollout in the ArgoCD UI or with `kubectl rollout status`.

---

## Extending the Repository

The following stacks are planned and have placeholder TODO markers:

- [ ] **Prometheus + Grafana** -- `applications/<env>/monitoring.yaml`
- [ ] **Loki** -- log aggregation
- [ ] **Tempo** -- distributed tracing backend
- [ ] **Jaeger** -- alternative tracing UI
- [ ] **cert-manager** -- TLS automation (bootstrap scaffold included)
- [ ] **NGINX Ingress** -- HTTP routing (bootstrap scaffold included)
- [ ] **Train Ticket** -- additional demo application

---

## Documentation

- [Architecture](docs/architecture.md)
- [Runbook](docs/runbook.md)

---

> **Security note:** This repository contains **no secrets**. Sensitive values
> (image pull secrets, database passwords, API keys) must be managed with
> Sealed Secrets, External Secrets Operator, or a compatible solution.
