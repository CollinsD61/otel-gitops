# Architecture — otel-gitops

## Overview

`otel-gitops` is a **GitOps repository** that declaratively manages the
deployment of the [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)
to Kubernetes using **ArgoCD** and **Helm**.

The core principle: **Git is the single source of truth**.
No manual `kubectl apply` or `helm install` commands are used for day-to-day
operations (except for the one-time bootstrap).

---

## System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DEVELOPER MACHINE                              │
│                                                                             │
│  $ git push origin main                                                     │
│  $ argocd app sync otel-demo-dev   (optional, manual trigger)               │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │ HTTPS push
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                          GITHUB (otel-gitops)                             │
│                                                                           │
│  main branch                                                              │
│  ├── bootstrap/root-app.yaml          ← manually applied once            │
│  ├── applications/dev/otel-demo.yaml  ← ArgoCD Application spec          │
│  ├── applications/staging/...                                             │
│  ├── applications/prod/...                                                │
│  └── environments/<env>/values.yaml  ← Helm overrides per env           │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ Poll / Webhook (every 3 min)
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        ArgoCD (namespace: argocd)                         │
│                                                                           │
│  root-app  ──── watches ────► applications/**/*.yaml                      │
│       │                                                                   │
│       └── creates ──────────► otel-demo-dev                              │
│                               otel-demo-staging                          │
│                               otel-demo-prod                             │
│                               ingress-nginx                              │
│                               cert-manager                               │
│                               metrics-server                             │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ Helm render + kubectl apply
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER (Kind / Cloud)                     │
│                                                                           │
│  namespace: otel-demo                                                     │
│  ├── frontend              (Deployment + Service + Ingress)               │
│  ├── cartService           (Deployment + Service)                         │
│  ├── checkoutService       (Deployment + Service)                         │
│  ├── productCatalogService (Deployment + Service)                         │
│  ├── recommendationService (Deployment + Service)                         │
│  ├── paymentService        (Deployment + Service)                         │
│  ├── shippingService       (Deployment + Service)                         │
│  ├── currencyService       (Deployment + Service)                         │
│  ├── adService             (Deployment + Service)                         │
│  ├── emailService          (Deployment + Service)                         │
│  ├── loadGenerator         (Deployment)                                   │
│  ├── kafka                 (StatefulSet)                                  │
│  ├── opentelemetry-collector (Deployment)                                 │
│  ├── jaeger                (Deployment + Service)                         │
│  ├── prometheus            (Deployment + Service)                         │
│  └── grafana               (Deployment + Service + Ingress)               │
│                                                                           │
│  namespace: ingress-nginx                                                 │
│  └── nginx ingress controller                                             │
│                                                                           │
│  namespace: cert-manager                                                  │
│  └── cert-manager + ClusterIssuers                                       │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## GitOps Workflow

### Normal Development Flow

```
1. Developer creates a feature branch
   $ git checkout -b feat/bump-otel-demo-chart-version

2. Edit environments/dev/values.yaml or applications/dev/otel-demo.yaml
   (e.g., bump targetRevision from 0.31.0 to 0.32.0)

3. Push and open a Pull Request
   $ git push origin feat/bump-otel-demo-chart-version

4. CI pipeline runs:
   - yamllint / kubeconform validation
   - helm lint / helm template dry-run
   - conftest / OPA policy checks (TODO)

5. Code review and merge to main

6. ArgoCD detects the change (poll interval: 3 min, or webhook)
   and auto-syncs the Application

7. Verify deployment:
   $ argocd app get otel-demo-dev
   $ kubectl rollout status deploy/otel-demo-frontend -n otel-demo
```

### Emergency Rollback

```
# Roll back to the previous ArgoCD revision
$ argocd app rollback otel-demo-prod <REVISION>

# Or via Helm (if ArgoCD manages the Helm release):
$ helm rollback otel-demo <REVISION> -n otel-demo
```

---

## App of Apps Pattern

The **App of Apps** pattern is the recommended approach for bootstrapping
multiple ArgoCD Applications.

```
bootstrap/root-app.yaml  (manually applied once)
         │
         ├── applications/dev/otel-demo.yaml
         ├── applications/staging/otel-demo.yaml
         ├── applications/prod/otel-demo.yaml
         ├── bootstrap/argocd/argocd-app.yaml         (ArgoCD self-manages itself)
         ├── bootstrap/ingress-nginx/ingress-nginx-app.yaml
         ├── bootstrap/cert-manager/cert-manager-app.yaml
         └── bootstrap/metrics-server/metrics-server-app.yaml
```

Benefits:
- A single `kubectl apply` bootstraps the entire cluster.
- Adding new applications only requires a Git commit.
- ArgoCD manages all Applications, including itself.

---

## Multi-Source Applications

Each otel-demo Application uses ArgoCD's **multi-source** feature:

```yaml
sources:
  # Source 1: The upstream Helm chart (not vendored)
  - repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
    chart: opentelemetry-demo
    targetRevision: "0.32.0"
    helm:
      valueFiles:
        - $values/environments/dev/values.yaml   # Reference to Source 2

  # Source 2: This repository (for values files only)
  - repoURL: https://github.com/<YOUR_ORG>/otel-gitops.git
    targetRevision: HEAD
    ref: values   # Alias used by $values above
```

This pattern lets us:
- Consume the upstream chart **without vendoring** it.
- Keep all customisation in this repository.
- Upgrade the chart version by changing one line in `applications/<env>/otel-demo.yaml`.

---

## Environment Promotion Strategy

```
dev  ──► staging  ──► prod
```

| Step | Action |
|------|--------|
| 1 | Develop and test in `dev` |
| 2 | Bump chart version in `staging` values |
| 3 | Run integration tests against `staging` |
| 4 | Promote to `prod` by updating `applications/prod/otel-demo.yaml` |
| 5 | Monitor Prometheus / Grafana dashboards after prod sync |

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| GitOps engine | ArgoCD 2.11+ | Continuous reconciliation |
| Package manager | Helm 3.14+ | Kubernetes application packaging |
| Cluster (local) | Kind | Local development cluster |
| Ingress | NGINX Ingress Controller | HTTP/HTTPS routing |
| TLS | cert-manager + Let's Encrypt | Automated certificate management |
| Metrics | metrics-server | HPA, `kubectl top` |
| Observability | OTel Demo (bundled Jaeger, Prometheus, Grafana) | Distributed tracing, metrics, dashboards |

---

## Future Additions

| Component | Purpose | Location |
|-----------|---------|----------|
| kube-prometheus-stack | Production monitoring | `applications/<env>/monitoring.yaml` |
| Loki | Log aggregation | `applications/<env>/loki.yaml` |
| Tempo | Distributed tracing backend | `applications/<env>/tempo.yaml` |
| Jaeger (dedicated) | Tracing UI | `applications/<env>/jaeger.yaml` |
| KEDA | Event-driven autoscaling | `bootstrap/keda/` |
| Sealed Secrets | Secret management | `bootstrap/sealed-secrets/` |
| Train Ticket | Additional demo app | `applications/<env>/train-ticket.yaml` |

---

## Security Considerations

- **No secrets in Git**: Use Sealed Secrets or External Secrets Operator.
- **RBAC**: ArgoCD projects should restrict which namespaces each Application can write to.
- **Pod Security Standards**: `baseline` enforced on all namespaces; `restricted` is the goal.
- **NetworkPolicy**: Default-deny-all with explicit allow rules (see `manifests/networkpolicy/`).
- **Image tags**: Always pin to immutable digest (`sha256:...`) in production; never use `latest`.
- **TLS everywhere**: cert-manager manages certificates; NGINX redirects HTTP to HTTPS.
