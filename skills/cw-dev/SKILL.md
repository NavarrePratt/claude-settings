---
name: cw-dev
description: Set up CoreWeave development environment and manage local CKS clusters. Use when installing development tools, configuring authentication, or starting/stopping local Kubernetes clusters.
allowed-tools:
  - Bash(cw dev:*)
  - Bash(cw version)
---

# CW Development Environment

Set up a CoreWeave development environment with required tools and manage local CKS (CoreWeave Kubernetes Service) clusters.

## When to Use This Skill

Use this skill when:
- Setting up a new development machine
- Installing CoreWeave development tools
- Configuring authentication for BSR/GitHub
- Starting or stopping a local CKS cluster for development
- Troubleshooting development environment issues

## Instructions

### Step 1: Understand the Request

Determine what the user needs:
- **Full setup**: Install all tools and configure auth
- **Auth only**: Just configure authentication
- **Specific tools**: Install only certain tools
- **CKS cluster**: Start or stop local Kubernetes cluster

### Step 2: Full Development Setup

**IMPORTANT: This installs software on the user's machine. Get confirmation before running.**

**Complete setup (auth + apps):**
```bash
cw dev init
```

This runs two stages:
1. `auths` - Authenticate with BSR and GitHub
2. `apps` - Install development tools

**Available tools installed:**
| Tool | Purpose |
|------|---------|
| devspace | Kubernetes development tool |
| direnv | Environment variable management |
| git | Version control |
| helm | Kubernetes package manager |
| homebrew | macOS package manager |
| kind | Kubernetes in Docker |
| kubectl | Kubernetes CLI |
| nix | Reproducible package manager |
| orbstack | Fast container runtime (macOS) |
| teleport | Secure infrastructure access |

### Step 3: Selective Setup

**Authentication only:**
```bash
cw dev init -s auths
```

**Apps only:**
```bash
cw dev init -s apps
```

**Specific apps:**
```bash
cw dev init -s apps -i helm,kubectl,kind
```

**Exclude certain apps:**
```bash
cw dev init -e orbstack,homebrew
```

### Step 4: Local CKS Cluster

For local Kubernetes development against CKS:

**Prerequisites:**
- Go installed
- SSH access to appenheimer repo
- Run `cw dev init` first

**Start cluster:**
```bash
cw dev cks --start
```

This:
1. Clones the appenheimer repository
2. Configures direnv
3. Starts the local CKS cluster

After starting:
```bash
cd appenheimer
source ~/.bashrc  # or ~/.zshrc
```

**Stop cluster:**
```bash
cw dev cks --stop
```

This runs `make clean` in the appenheimer directory.

### Step 5: Verify Setup

After setup, verify tools are working:

```bash
# Check installed tools
kubectl version --client
helm version
kind version

# Check auth
gh auth status
```

## Common Workflows

### New Engineer Setup

```bash
# 1. Install cw CLI
gh api -H 'Accept: application/vnd.github.v3.raw' \
   "repos/coreweave/cw-eng-cli/contents/scripts/install.sh" | bash

# 2. Run full dev setup
cw dev init

# 3. Start local cluster (if needed)
cw dev cks --start
```

### Minimal Setup (Just Kubernetes Tools)

```bash
cw dev init -s apps -i kubectl,helm,kind
```

### Reset Authentication

If auth is broken:
```bash
rm -rf ~/.cw/cli/gh.json
cw dev init -s auths
```

## Troubleshooting

**Tool installation fails:**
- Check if Homebrew is working: `brew doctor`
- Check permissions on install directories
- Try installing individual tools: `cw dev init -s apps -i <tool>`

**CKS cluster won't start:**
- Ensure Go is installed: `go version`
- Check SSH access to appenheimer
- Verify Docker/OrbStack is running

**Auth issues:**
```bash
rm -rf ~/.cw/cli/gh.json
cw dev init -s auths
```

**View logs:**
```bash
cat ~/.cw/cli/cli.log
```

## Platform Notes

**macOS:**
- OrbStack is recommended over Docker Desktop
- Homebrew is used for package management

**Linux:**
- Some tools may use Nix instead of Homebrew
- OrbStack is macOS-only

**CI/CD:**
- Set `CI=true` to disable interactive prompts
- Use `GITHUB_TOKEN` env var for auth
