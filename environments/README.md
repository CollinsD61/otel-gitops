# environments/

This directory contains **Helm values files** for each environment.

Each environment sub-directory holds one `values.yaml` file (and optionally
additional override files) that customize the OpenTelemetry Demo Helm chart
for that environment.

## Structure

```
environments/
+-- dev/
|   +-- values.yaml      # Low resources, single replicas, NodePort
+-- staging/
|   +-- values.yaml      # Medium resources, 2 replicas, Ingress
+-- prod/
    +-- values.yaml      # Full resources, HPA, TLS, LoadBalancer
```

## Principle

- Values files contain **only overrides** from the chart defaults.
- Secrets are **never** stored here. Use Sealed Secrets or External Secrets.
- Each file is heavily commented to explain every override.

## Adding a new application

When adding a new Helm-based application (e.g., Grafana), create:
  `environments/<env>/<app-name>-values.yaml`

## TODO

- [ ] Add `environments/<env>/monitoring-values.yaml` for Prometheus + Grafana
- [ ] Add `environments/<env>/loki-values.yaml`
- [ ] Add `environments/<env>/tempo-values.yaml`
