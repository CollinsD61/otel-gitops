# bootstrap/argocd/

ArgoCD **self-management** manifests.

By having ArgoCD manage its own installation via GitOps, any upgrade or
configuration change to ArgoCD is done through a pull request — not a manual
`helm upgrade` command.

## Files

| File | Purpose |
|------|---------|
| `argocd-app.yaml` | ArgoCD Application that installs ArgoCD itself via Helm |
| `argocd-values.yaml` | Helm values for the ArgoCD chart |

## TODO

- [ ] Enable ArgoCD HA (high-availability) mode for production
- [ ] Configure OIDC / SSO (Dex or external IdP)
- [ ] Set up RBAC policies (`argocd-rbac-cm`)
- [ ] Enable ArgoCD notifications (Slack, PagerDuty)
- [ ] Tune resource requests/limits for production load
