---
name: cw-explore
description: Discover what the cw CLI can do for you. Use when learning about cw CLI capabilities, exploring available components and archetypes, or deciding what features to add to a repository.
allowed-tools:
  - Bash(cw version)
  - Bash(cw:*)
---

# CW CLI Explorer

Interactive discovery skill for the CoreWeave engineering CLI. Helps you understand what's available and what might benefit your project.

## When to Use This Skill

Use this skill when:
- You're new to the cw CLI and want to learn what it offers
- You want to see what components are available to add to a repo
- You want to explore archetype options for a new project
- You need to understand what a specific component or archetype does

## Instructions

### Step 1: Check CLI Status

First, verify the CLI is installed and check the version:

```bash
cw version
```

If not installed, guide the user to install it:
```bash
gh api -H 'Accept: application/vnd.github.v3.raw' \
   "repos/coreweave/cw-eng-cli/contents/scripts/install.sh" | bash
```

### Step 2: Understand User Context

Ask the user about their context:
1. Are they exploring for a new project or an existing repo?
2. What kind of project (Go service, Python app, general)?
3. What problems are they trying to solve?

### Step 3: Show Relevant Options

Based on context, present the most relevant options:

**For new projects:**
- Explain available archetypes (blank-repo, go-http-service)
- Show how to preview: `cw scaffold generate -a`

**For existing repos:**
- List available component groups and what they solve
- Recommend based on what the repo is missing

**Component Categories:**

| Category | Components | Solves |
|----------|------------|--------|
| backstage | catalog-yaml-* | Service catalog registration |
| github | codeowners | Code ownership and review routing |
| github | workflow-megalinter | Comprehensive linting |
| github | workflow-renovate | Dependency updates |
| github | workflow-claude-review-prs | AI PR review |
| github | workflow-close-stale-prs | PR hygiene |
| helm | chart-basic | Kubernetes deployment |

### Step 4: Deep Dive

If the user wants more info about a specific component or archetype:

```bash
cw scaffold info -c   # For component info
cw scaffold info -a   # For archetype info
```

### Step 5: Next Steps

Based on what they want to do, guide them to the appropriate skill:
- `/cw-repo` for creating a new repository
- `/cw-scaffold` for adding components or generating archetypes
- `/cw-dev` for setting up their development environment

## Quick Reference

**Check what's available:**
```bash
cw scaffold generate -c  # Interactive component selection
cw scaffold generate -a  # Interactive archetype selection
```

**Get component details:**
```bash
cw scaffold info -c --version latest
```

**Check your current setup:**
```bash
cw version
ls ~/.cw/cli/
```
