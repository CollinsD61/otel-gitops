# manifests/

This directory contains raw **Kubernetes manifests** that are not managed by
the Helm chart — typically cluster-scoped or cross-cutting resources.

## When to add a manifest here

- The resource is not part of any Helm chart.
- You need fine-grained control over a resource the chart creates (e.g., a
  custom HPA or NetworkPolicy that overrides chart defaults).
- The resource is cluster-scoped (e.g., a Namespace, ClusterRole).

## Structure

```
manifests/
+-- namespaces/       # Namespace definitions (labels, annotations)
+-- networkpolicy/    # NetworkPolicy resources for micro-segmentation
+-- hpa/              # Horizontal Pod Autoscaler manifests
+-- ingress/          # Ingress rules (if not managed by Helm values)
```

## How ArgoCD uses these manifests

These manifests are referenced by additional ArgoCD Applications (or by adding
a `path` source to the existing otel-demo Application) so they are
continuously reconciled.

## TODO

- [ ] Add PodDisruptionBudget (PDB) manifests for critical services
- [ ] Add ResourceQuota per namespace
- [ ] Add LimitRange per namespace
- [ ] Add RBAC (ServiceAccount, Role, RoleBinding) for each service
