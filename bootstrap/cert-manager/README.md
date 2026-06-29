# bootstrap/cert-manager/

ArgoCD Application that installs
[cert-manager](https://cert-manager.io/) for automated TLS certificate
management (Let's Encrypt, internal CA, etc.).

## Files

| File | Purpose |
|------|---------|
| `cert-manager-app.yaml` | ArgoCD Application for cert-manager |
| `clusterissuer-letsencrypt.yaml` | ClusterIssuer for Let's Encrypt (staging + prod) |

## Notes

- cert-manager CRDs must be installed **before** any `Certificate` or
  `ClusterIssuer` resources.
- The ArgoCD Application uses `ServerSideApply=true` to handle CRD sizes.

## TODO

- [ ] Configure production `ClusterIssuer` with your email address
- [ ] Configure DNS-01 challenge for wildcard certificates
- [ ] Add support for internal CA (for on-prem / air-gapped)
- [ ] Test certificate renewal automation
