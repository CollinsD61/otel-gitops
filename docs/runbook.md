# Runbook — otel-gitops

This runbook contains common operational commands for managing the
`otel-gitops` cluster.

> **Tip**: Alias long commands in your shell profile.  Examples are included
> at the bottom of this document.

---

## Table of Contents

- [Cluster Management (Kind)](#cluster-management-kind)
- [ArgoCD Operations](#argocd-operations)
- [Helm Operations](#helm-operations)
- [kubectl Cheatsheet](#kubectl-cheatsheet)
- [Troubleshooting](#troubleshooting)
- [Useful Aliases](#useful-aliases)

---

## Cluster Management (Kind)

### Create a local cluster

```bash
# Create cluster using the default configuration
kind create cluster --name otel-gitops

# Create with a custom config (recommended for port mappings)
kind create cluster --name otel-gitops --config bootstrap/kind-config.yaml

# List clusters
kind get clusters

# Switch kubectl context to Kind cluster
kubectl config use-context kind-otel-gitops

# Get cluster info
kubectl cluster-info --context kind-otel-gitops
```

### Delete the local cluster

```bash
kind delete cluster --name otel-gitops
```

### Load a local Docker image into Kind

```bash
# Build image locally
docker build -t my-service:local .

# Load into Kind (avoids Docker Hub pull)
kind load docker-image my-service:local --name otel-gitops
```

### Kind node shell (debugging)

```bash
# List Kind nodes (Docker containers)
docker ps --filter "label=io.x-k8s.kind.cluster=otel-gitops"

# Exec into a Kind node
docker exec -it otel-gitops-control-plane bash
```

---

## ArgoCD Operations

### Initial setup

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl rollout status deploy/argocd-server -n argocd --timeout=5m

# Get initial admin password
argocd admin initial-password -n argocd

# Port-forward the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080 (accept self-signed cert warning)

# Login via CLI
argocd login localhost:8080 --insecure --username admin

# Change the default admin password immediately
argocd account update-password
```

### Bootstrap the cluster

```bash
# Apply the root App of Apps (one-time manual step)
kubectl apply -f bootstrap/root-app.yaml

# Watch Applications being created
watch argocd app list
```

### Application management

```bash
# List all Applications
argocd app list

# Get detailed status of an Application
argocd app get otel-demo-dev
argocd app get otel-demo-staging
argocd app get otel-demo-prod

# Sync an Application manually (force ArgoCD to reconcile now)
argocd app sync otel-demo-dev
argocd app sync otel-demo-dev --prune   # Also prune orphaned resources

# Sync with a dry-run (preview what will change)
argocd app diff otel-demo-dev

# Sync all Applications
argocd app sync --selector app.kubernetes.io/part-of=otel-gitops

# Force hard refresh (re-fetch from Git and Helm repo)
argocd app get otel-demo-dev --hard-refresh

# Watch sync status in real time
argocd app wait otel-demo-dev --sync --timeout 300
```

### Rollback

```bash
# List revision history
argocd app history otel-demo-prod

# Roll back to a specific revision
argocd app rollback otel-demo-prod <REVISION>

# Disable auto-sync temporarily (e.g., during an incident)
argocd app set otel-demo-prod --sync-policy none

# Re-enable auto-sync after the incident
argocd app set otel-demo-prod \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### ArgoCD CLI — useful queries

```bash
# List all out-of-sync Applications
argocd app list -o json | jq '.[] | select(.status.sync.status != "Synced") | .metadata.name'

# Show all resources managed by an Application
argocd app resources otel-demo-prod

# Delete an Application (without deleting cluster resources)
argocd app delete otel-demo-dev --cascade=false

# Delete an Application AND its cluster resources
argocd app delete otel-demo-dev --cascade=true
```

---

## Helm Operations

### Add the OpenTelemetry Helm repository

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

### Inspect the chart

```bash
# List available chart versions
helm search repo open-telemetry/opentelemetry-demo --versions

# Show default values
helm show values open-telemetry/opentelemetry-demo --version 0.32.0

# Show chart README
helm show readme open-telemetry/opentelemetry-demo

# Render templates locally (dry-run with dev values)
helm template otel-demo open-telemetry/opentelemetry-demo \
  --version 0.32.0 \
  --values environments/dev/values.yaml \
  --namespace otel-demo \
  > /tmp/otel-demo-dev-rendered.yaml

# Lint values files
helm lint open-telemetry/opentelemetry-demo \
  --version 0.32.0 \
  --values environments/prod/values.yaml
```

### Manual install / upgrade (development only)

```bash
# Install manually (bypass ArgoCD — use only for debugging)
helm install otel-demo open-telemetry/opentelemetry-demo \
  --version 0.32.0 \
  --namespace otel-demo \
  --create-namespace \
  --values environments/dev/values.yaml

# Upgrade
helm upgrade otel-demo open-telemetry/opentelemetry-demo \
  --version 0.32.0 \
  --namespace otel-demo \
  --values environments/dev/values.yaml

# List installed releases
helm list -n otel-demo
helm list -A   # All namespaces

# Get status of a release
helm status otel-demo -n otel-demo

# Uninstall (WARNING: deletes all resources)
helm uninstall otel-demo -n otel-demo

# Helm rollback
helm rollback otel-demo <REVISION> -n otel-demo

# View rollback history
helm history otel-demo -n otel-demo
```

---

## kubectl Cheatsheet

### Namespace operations

```bash
# List all namespaces
kubectl get ns

# Describe namespace (check labels, annotations, resource quotas)
kubectl describe ns otel-demo

# Set default namespace for current context
kubectl config set-context --current --namespace=otel-demo
```

### Pod debugging

```bash
# List all pods in otel-demo
kubectl get pods -n otel-demo
kubectl get pods -n otel-demo -o wide     # Include node and IP
kubectl get pods -n otel-demo -w          # Watch in real time

# Describe a failing pod
kubectl describe pod <POD_NAME> -n otel-demo

# Stream logs from a pod
kubectl logs <POD_NAME> -n otel-demo -f

# Stream logs from a specific container
kubectl logs <POD_NAME> -c <CONTAINER_NAME> -n otel-demo -f

# Previous container logs (after a crash)
kubectl logs <POD_NAME> -n otel-demo --previous

# Execute a shell inside a pod (debugging)
kubectl exec -it <POD_NAME> -n otel-demo -- /bin/sh

# Run a temporary debug pod
kubectl run debug --image=busybox:latest --rm -it --restart=Never -n otel-demo -- sh
```

### Deployment operations

```bash
# List deployments
kubectl get deployments -n otel-demo

# Check rollout status
kubectl rollout status deploy/otel-demo-frontend -n otel-demo

# Rollout history
kubectl rollout history deploy/otel-demo-frontend -n otel-demo

# Undo last rollout
kubectl rollout undo deploy/otel-demo-frontend -n otel-demo

# Scale a deployment manually (will be overridden by ArgoCD selfHeal)
kubectl scale deploy/otel-demo-frontend --replicas=3 -n otel-demo

# Restart a deployment (rolling restart)
kubectl rollout restart deploy/otel-demo-frontend -n otel-demo
```

### Service and networking

```bash
# List services
kubectl get svc -n otel-demo

# Port-forward to the frontend (local access without Ingress)
kubectl port-forward svc/otel-demo-frontend 8080:8080 -n otel-demo
# Open: http://localhost:8080

# Port-forward to Grafana
kubectl port-forward svc/otel-demo-grafana 3000:80 -n otel-demo
# Open: http://localhost:3000

# Port-forward to Jaeger UI
kubectl port-forward svc/otel-demo-jaeger-query 16686:16686 -n otel-demo
# Open: http://localhost:16686

# Port-forward to Prometheus
kubectl port-forward svc/otel-demo-prometheus 9090:9090 -n otel-demo
# Open: http://localhost:9090

# Inspect an Ingress
kubectl get ingress -n otel-demo
kubectl describe ingress otel-demo-frontend -n otel-demo
```

### Resource metrics

```bash
# Pod resource usage (requires metrics-server)
kubectl top pods -n otel-demo
kubectl top pods -n otel-demo --sort-by=cpu

# Node resource usage
kubectl top nodes

# HPA status
kubectl get hpa -n otel-demo
kubectl describe hpa frontend-hpa -n otel-demo
```

### Secrets and ConfigMaps

```bash
# List secrets (do not print values)
kubectl get secrets -n otel-demo

# Describe a secret (shows keys but not values)
kubectl describe secret <SECRET_NAME> -n otel-demo

# Decode a secret value (use carefully — never log in CI)
kubectl get secret <SECRET_NAME> -n otel-demo \
  -o jsonpath='{.data.<KEY>}' | base64 -d

# List ConfigMaps
kubectl get cm -n otel-demo
```

### Events and troubleshooting

```bash
# Show recent events in a namespace
kubectl get events -n otel-demo --sort-by='.lastTimestamp'

# Show only warning events
kubectl get events -n otel-demo --field-selector type=Warning

# Check NetworkPolicy
kubectl get networkpolicy -n otel-demo
kubectl describe networkpolicy default-deny-all -n otel-demo
```

---

## Troubleshooting

### ArgoCD Application is OutOfSync

```bash
# 1. Check what's different
argocd app diff otel-demo-dev

# 2. Hard-refresh to ensure ArgoCD has latest Git state
argocd app get otel-demo-dev --hard-refresh

# 3. Sync manually
argocd app sync otel-demo-dev

# 4. Check ArgoCD repo-server logs for Helm errors
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -f
```

### Pod is CrashLoopBackOff

```bash
# 1. Check events
kubectl describe pod <POD_NAME> -n otel-demo

# 2. Check previous container logs
kubectl logs <POD_NAME> -n otel-demo --previous

# 3. Check resource limits (OOMKilled?)
kubectl get pod <POD_NAME> -n otel-demo -o json | jq '.status.containerStatuses[].lastState'

# 4. Increase memory limit in environments/<env>/values.yaml and push to Git
```

### Ingress not routing traffic

```bash
# 1. Check Ingress resource
kubectl describe ingress otel-demo-frontend -n otel-demo

# 2. Check NGINX Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# 3. Check certificate status
kubectl describe certificate otel-demo-tls -n otel-demo
kubectl describe certificaterequest -n otel-demo

# 4. Check ClusterIssuer
kubectl describe clusterissuer letsencrypt-staging
```

### HPA not scaling

```bash
# 1. Check HPA status and events
kubectl describe hpa frontend-hpa -n otel-demo

# 2. Verify metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
kubectl top pods -n otel-demo

# 3. Check resource requests are set (HPA requires them)
kubectl get deploy otel-demo-frontend -n otel-demo -o json \
  | jq '.spec.template.spec.containers[].resources'
```

### cert-manager not issuing certificate

```bash
# 1. Check Certificate object
kubectl describe certificate otel-demo-tls -n otel-demo

# 2. Check CertificateRequest
kubectl get certificaterequest -n otel-demo
kubectl describe certificaterequest -n otel-demo

# 3. Check ACME challenge (HTTP-01)
kubectl get challenge -n cert-manager
kubectl describe challenge -n cert-manager

# 4. Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager -f

# 5. Ensure the Ingress is accessible from the internet (for HTTP-01)
# For local Kind: use staging issuer and skip TLS, or set up a tunnel (ngrok)
```

---

## Useful Aliases

Add these to your `~/.bashrc` or `~/.zshrc`:

```bash
# kubectl shortcuts
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs -f'
alias ke='kubectl exec -it'

# Namespace shortcuts
alias koteldemo='kubectl -n otel-demo'
alias kargocd='kubectl -n argocd'
alias knginx='kubectl -n ingress-nginx'
alias kcert='kubectl -n cert-manager'

# ArgoCD shortcuts
alias aapp='argocd app'
alias alist='argocd app list'
alias async='argocd app sync'
alias adiff='argocd app diff'

# Port-forwards
alias pf-argocd='kubectl port-forward svc/argocd-server -n argocd 8080:443 &'
alias pf-frontend='kubectl port-forward svc/otel-demo-frontend -n otel-demo 8080:8080 &'
alias pf-grafana='kubectl port-forward svc/otel-demo-grafana -n otel-demo 3000:80 &'
alias pf-jaeger='kubectl port-forward svc/otel-demo-jaeger-query -n otel-demo 16686:16686 &'
alias pf-prometheus='kubectl port-forward svc/otel-demo-prometheus -n otel-demo 9090:9090 &'

# Kind shortcuts
alias kc='kind create cluster --name otel-gitops'
alias kd-cluster='kind delete cluster --name otel-gitops'
alias kctx='kubectl config use-context kind-otel-gitops'
```
