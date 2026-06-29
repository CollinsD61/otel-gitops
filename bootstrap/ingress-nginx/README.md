# bootstrap/ingress-nginx/

ArgoCD Application that installs the
[NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
into the cluster.

## Files

| File | Purpose |
|------|---------|
| `ingress-nginx-app.yaml` | ArgoCD Application for NGINX Ingress |
| `ingress-nginx-values.yaml` | Helm values for the ingress-nginx chart |

## Notes

- The Ingress Controller is deployed into the `ingress-nginx` namespace.
- For Kind clusters, use `hostPort` or `NodePort` instead of `LoadBalancer`.
- For cloud clusters, set `controller.service.type: LoadBalancer`.

## TODO

- [ ] Configure default SSL certificate
- [ ] Enable ModSecurity WAF (production)
- [ ] Set rate-limiting annotations on Ingress resources
- [ ] Enable metrics scraping for Prometheus
