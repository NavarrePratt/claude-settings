# CW CLI - CoreWeave Engineering CLI

The `cw` CLI is CoreWeave's command-line tool for engineering workflows, built by the CI Build Services team. It also includes `rs` (RepoSmith) for organization-wide repository analysis.

## Quick Reference

| Task | Command |
|------|---------|
| Create new repo | `cw repo create` |
| Add component to repo | `cw scaffold generate -c` |
| Generate archetype locally | `cw scaffold generate -a` |
| Validate templates | `cw scaffold validate <path>` |
| Setup dev environment | `cw dev init` |
| Start local CKS cluster | `cw dev cks --start` |
| Stop local CKS cluster | `cw dev cks --stop` |
| Update CLI | `cw update` |
| Check version | `cw version` |

## Commands

### Repository Creation

```bash
cw repo create
```

Interactive command that:
1. Prompts for repository name, description, team, visibility
2. Selects archetype template (blank-repo, go-http-service)
3. Configures team permissions and branch protection
4. Registers in Backstage catalog
5. Optionally creates PR for review

### Scaffolding

**Add component to existing repo:**
```bash
cw scaffold generate -c
# or with specific version
cw scaffold generate -c --version v2.5.0
```

**Generate archetype locally:**
```bash
cw scaffold generate -a
```

**Non-interactive with config file:**
```yaml
# config.yaml
componentGroup: github
componentName: codeowners
inputs:
  github_team_name: platform
```
```bash
cw scaffold generate --config config.yaml
```

**Validate templates:**
```bash
cw scaffold validate <path> -c  # component
cw scaffold validate <path> -a  # archetype
```

### Development Environment

**Full setup:**
```bash
cw dev init
```

**Authentication only:**
```bash
cw dev init -s auths
```

**Specific apps only:**
```bash
cw dev init -s apps -i helm,kubectl,kind
```

**Exclude apps:**
```bash
cw dev init -e orbstack,homebrew
```

**Available apps:** devspace, direnv, git, helm, homebrew, kind, kubectl, nix, orbstack, teleport

**Local CKS cluster:**
```bash
cw dev cks --start   # Start cluster
cw dev cks --stop    # Stop cluster
```

### CLI Updates

```bash
cw update            # Update to latest
cw update --check    # Check for updates
cw update -v v1.15.0 # Specific version
cw update -y         # Skip confirmation
```

## Available Components

| Group | Component | Description |
|-------|-----------|-------------|
| backstage | catalog-yaml-component | Backstage Component object |
| backstage | catalog-yaml-location | Backstage Location object |
| backstage | catalog-yaml-system | Backstage System object |
| github | codeowners | CODEOWNERS file |
| github | workflow-claude-review-prs | AI-powered PR review |
| github | workflow-close-stale-prs | Auto-close stale PRs |
| github | workflow-megalinter | Comprehensive linting |
| github | workflow-renovate | Dependency management |
| helm | chart-basic | Helm chart with CoreWeave best practices |

## Available Archetypes

| Archetype | Description |
|-----------|-------------|
| blank-repo | Minimal starter with CODEOWNERS and Backstage |
| go-http-service | Production-ready Go HTTP service with CI/CD |

## Authentication

**Interactive (default):**
```
1. Visit: https://github.com/login/device
2. Enter code: XXXX-XXXX
3. Authorize the application
```

Tokens stored at `~/.cw/cli/gh.json`. Auto-refreshes every 8 hours.

**CI/CD:**
```bash
export GITHUB_TOKEN="ghp_your_token_here"
```

**Re-authenticate:**
```bash
rm -rf ~/.cw/cli/gh.json
cw update  # Triggers fresh auth
```

**Required scopes:** read:user, read:org, repo, user:email, workflow

## RepoSmith (rs) Commands

Organization-wide repository analysis tool.

```bash
rs config init --org coreweave    # Initialize config
rs config show                     # Show config
rs repo init                       # Clone repos
rs repo inspect --update-config   # Analyze repos
rs repo list                       # List repos
rs repo report --type codeowners  # Generate reports
```

**Report types:** codeowners, actions, languages, documentation, activity

## Configuration

**Config directory:** `~/.cw/cli/`
- `gh.json` - GitHub tokens
- `cached-templates/` - Template cache
- `cli.log` - Logs

**Clear cache:**
```bash
rm -rf ~/.cw/cli/cached-templates/
```

**Test local templates:**
```bash
cw scaffold generate -c --path /path/to/local/repo-templates
```

## Autonomy Guidelines for Claude

**Execute autonomously:**
- `cw version` - Check version
- `cw update --check` - Check for updates
- `cw scaffold validate` - Validate templates
- `rs repo list` - List repos
- `rs config show` - Show config

**Require user confirmation:**
- `cw repo create` - Creates GitHub repository
- `cw scaffold generate` - Generates files
- `cw dev init` - Installs software
- `cw dev cks` - Starts/stops clusters
- `cw update` - Updates CLI binary
- `rs repo init` - Clones repositories

## Troubleshooting

**View logs:**
```bash
cat ~/.cw/cli/cli.log
```

**Auth issues:**
```bash
rm -rf ~/.cw/cli/gh.json
# Re-run command to re-authenticate
```

**Template issues:**
```bash
rm -rf ~/.cw/cli/cached-templates/
```

**In CI (disable update checks):**
```bash
export CI=true
```

## Getting Help

- Slack: #ci-build-services
- CBS Helpdesk: JIRA Service Desk portal
- Source: github.com/coreweave/cw-eng-cli
