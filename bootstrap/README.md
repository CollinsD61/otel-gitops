# bootstrap/

This directory contains everything needed to **bootstrap** ArgoCD and the
cluster's foundational infrastructure using the **App of Apps** pattern.

## How it works

1. You manually `kubectl apply -f bootstrap/root-app.yaml` **once**.
2. ArgoCD picks up `root-app.yaml` and creates child Applications for each
   entry in `bootstrap/argocd/`, `bootstrap/ingress-nginx/`, etc.
3. From that point forward, all changes are made via Git — ArgoCD continuously
   reconciles the cluster.

## Sub-directories

| Directory | Purpose |
|-----------|---------|
| `argocd/` | ArgoCD self-managed install (HA values, RBAC) |
| `ingress-nginx/` | NGINX Ingress Controller |
| `metrics-server/` | Kubernetes Metrics Server |
| `cert-manager/` | cert-manager for TLS certificate automation |

## Adding new bootstrap components

1. Create a new sub-directory (e.g., `bootstrap/sealed-secrets/`).
2. Add an ArgoCD `Application` manifest inside it.
3. Reference it from `root-app.yaml` by adding a new source path.
4. Commit and push -- ArgoCD will pick it up automatically.

## TODO

- [ ] Add `bootstrap/prometheus-stack/` for monitoring
- [ ] Add `bootstrap/loki/` for log aggregation
- [ ] Add `bootstrap/tempo/` for distributed tracing
- [ ] Add `bootstrap/sealed-secrets/` or `bootstrap/external-secrets/`
