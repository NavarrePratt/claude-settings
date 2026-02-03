---
name: cw-repo
description: Guide through creating a new CoreWeave GitHub repository with proper setup. Use when creating new repos with standardized templates, team permissions, and Backstage integration.
allowed-tools:
  - Bash(cw repo:*)
  - Bash(gh:*)
---

# CW Repository Creation

Guide users through creating new GitHub repositories using the cw CLI. Ensures proper setup with team permissions, branch protection, and Backstage integration.

## When to Use This Skill

Use this skill when:
- Creating a new GitHub repository in the CoreWeave organization
- Setting up a new project with proper configuration from the start
- Need a repository with Backstage catalog integration

## Instructions

### Step 1: Pre-flight Checks

Before starting, verify:

1. **CLI installed:**
   ```bash
   cw version
   ```

2. **Authenticated:**
   - The command will prompt for GitHub auth if needed
   - Uses OAuth device flow

3. **Gather requirements:**
   - Repository name (kebab-case recommended)
   - Short description
   - Owning team
   - Visibility (private/public)
   - Archetype (blank-repo or go-http-service)

### Step 2: Present Archetype Options

Help the user choose the right archetype:

| Archetype | Best For |
|-----------|----------|
| blank-repo | General projects, scripts, documentation, non-Go projects |
| go-http-service | Go HTTP/gRPC services with CI/CD, Docker, Kubernetes deployment |

### Step 3: Guide Through Creation

**IMPORTANT: This command creates a real GitHub repository. Always get user confirmation before running.**

Show the user what will happen:
```
cw repo create will:
1. Create GitHub repository: coreweave/<name>
2. Apply archetype template
3. Configure team permissions
4. Set up branch protection (main)
5. Register in Backstage catalog
6. Optionally create a PR for review
```

Once confirmed, run:
```bash
cw repo create
```

The command is interactive and will prompt for:
- Repository name
- Description
- Team selection
- Visibility
- Archetype selection

### Step 4: Post-Creation

After repository creation:

1. **Clone the repo:**
   ```bash
   gh repo clone coreweave/<repo-name>
   cd <repo-name>
   ```

2. **Verify setup:**
   - Check branch protection rules
   - Verify team access
   - Confirm Backstage registration

3. **Next steps:**
   - Add additional components with `/cw-scaffold`
   - Set up local development with `/cw-dev`

## What Gets Created

**blank-repo archetype:**
- `.github/CODEOWNERS`
- `catalog.yaml` (Backstage)
- Basic README.md
- Branch protection on main

**go-http-service archetype:**
- All of blank-repo, plus:
- Go project structure
- Dockerfile
- Kubernetes manifests
- CI/CD workflows
- Makefile

## Troubleshooting

**Auth issues:**
```bash
rm -rf ~/.cw/cli/gh.json
cw repo create  # Re-authenticates
```

**Permission denied:**
- Ensure you have org access
- Check team membership
- Verify GitHub token scopes

**Template errors:**
```bash
rm -rf ~/.cw/cli/cached-templates/
cw repo create  # Re-fetches templates
```
