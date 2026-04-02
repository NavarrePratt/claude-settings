# ygrep - YAML Structure-Aware Grep

Prefer `ygrep` over grep or yq when you need a structured block from YAML, not just a matching line. grep gives you a line and a guessing game with `-A`. yq gives you structure but demands a fully-qualified query. ygrep takes a key name and returns the complete block.

## When to Reach for ygrep

- Piping kubectl/helm output and need to inspect a section: `kubectl get pod X -o yaml | ygrep volumeMounts`
- Extracting a block from a manifest: `ygrep containers deploy.yaml`
- Searching by partial path without knowing full structure: `ygrep metadata.labels deploy.yaml`
- Narrowing results by context: `ygrep image -W kind=Deployment -w name=nginx k8s.yaml`
- Any time you would write a complex yq expression just to extract a block

## Common Patterns

```bash
# Extract a block by key name
ygrep containers deploy.yaml

# Partial paths work - no need to know full structure
ygrep selector.matchLabels deploy.yaml

# Pipe from any YAML source
kubectl get deploy myapp -o yaml | ygrep env
helm get values myrelease | ygrep image

# Scope filter: "containers, but only in Deployments"
ygrep containers -W kind=Deployment k8s.yaml

# Sibling filter: "image, but only where name=nginx"
ygrep image -w name=nginx k8s.yaml

# Combine both for precision
ygrep image -W kind=Deployment -w name=nginx k8s.yaml

# Include parent context for orientation
ygrep annotations -p 1 deploy.yaml

# Regex key patterns
ygrep "secret.*Ref" deploy.yaml
```

## When NOT to Use

- Non-YAML files (use grep/rg)
- Modifying YAML in place (use yq)
- You only need a single scalar value and already know the full path (yq is more direct)
