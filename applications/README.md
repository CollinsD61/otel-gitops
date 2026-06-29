# applications/

This directory contains one ArgoCD `Application` manifest per environment per
application.

## Structure

```
applications/
+-- dev/
|   +-- otel-demo.yaml       # ArgoCD Application for dev
+-- staging/
|   +-- otel-demo.yaml       # ArgoCD Application for staging
+-- prod/
    +-- otel-demo.yaml       # ArgoCD Application for prod
```

## Design

- Each file declares **one** ArgoCD `Application`.
- The `root-app.yaml` in `bootstrap/` watches this directory recursively and
  creates all Applications automatically.
- Helm values are sourced from `environments/<env>/values.yaml` in this
  same repository — no secrets are stored here.

## Adding a new application

1. Create `applications/<env>/<app-name>.yaml`.
2. Point its `source.repoURL` and `source.chart` at the upstream Helm chart.
3. Create matching `environments/<env>/<app-name>-values.yaml`.
4. Commit and push — ArgoCD reconciles automatically.

## TODO

- [ ] Add `applications/<env>/monitoring.yaml` (Prometheus + Grafana)
- [ ] Add `applications/<env>/loki.yaml`
- [ ] Add `applications/<env>/tempo.yaml`
- [ ] Add `applications/<env>/jaeger.yaml`
- [ ] Add `applications/<env>/train-ticket.yaml`
