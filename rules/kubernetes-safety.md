# Kubernetes Safety

## Assume Large Clusters

Always assume Kubernetes clusters are very large. Never run broadly-scoped queries.

**NEVER do:**
- `kubectl get pods --all-namespaces` or `-A`
- `kubectl get <resource> --all-namespaces`
- Any cluster-wide resource listing without namespace scoping
- Piping all-namespace output through scripts that process every object

**ALWAYS do:**
- Scope queries to a specific namespace: `kubectl get pods -n <namespace>`
- When you need cluster-wide info, query specific resource types on specific nodes or namespaces
- Ask the user which namespace to target if unclear
- Use `kubectl get nodes` (node count is bounded and safe) but avoid fetching all pods/deployments/etc across the cluster

This applies to customer clusters, shared environments, and any cluster accessed via provided kubeconfigs. The cost of an unscoped query on a large cluster ranges from slow responses to API server overload.
