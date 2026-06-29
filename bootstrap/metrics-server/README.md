# bootstrap/metrics-server/

ArgoCD Application that installs the
[Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

The Metrics Server is a prerequisite for:
- `kubectl top pods` / `kubectl top nodes`
- Horizontal Pod Autoscaler (HPA) based on CPU / memory

## Files

| File | Purpose |
|------|---------|
| `metrics-server-app.yaml` | ArgoCD Application for metrics-server |

## Notes

- For Kind clusters, `--kubelet-insecure-tls` is required (see values file).
- For production clusters, TLS should be properly configured.

## TODO

- [ ] Replace with Prometheus Adapter for custom-metrics HPA in production
- [ ] Tune `requestheader` certificates for mutual TLS
