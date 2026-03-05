# Centralized MCP Servers

Centralized MCP server management via docker-compose. Replaces per-session
Docker container spawning with one persistent container per MCP server, shared
across all Claude Code sessions via SSE transport.

## Quick Start

```bash
cd ~/.claude/mcp-servers
cp .env.example .env   # Fill in tokens
docker compose up -d
```

## Management

```bash
docker compose up -d                          # Start all services
docker compose down                           # Stop all services
docker compose ps                             # Check status
docker compose logs -f [service]              # Tail logs (e.g. slack-mcp, grafana-mcp)
docker compose pull && docker compose up -d   # Update images
```

## Token Rotation

1. Edit `.env` with new token values
2. `docker compose restart [service]`
3. For `SLACK_MCP_API_KEY` changes: also update the bearer token in Claude MCP
   config (`claude mcp remove --scope user slack` then re-add with new header)

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Connection refused | `docker compose ps` - verify containers are running and ports are bound |
| Auth errors (401) | Verify `SLACK_MCP_API_KEY` in `.env` matches the `Authorization` header in Claude MCP config |
| Blank responses | `docker compose logs -f slack-mcp` - look for upstream API errors |
| Cache stale | Remove contents of `data/slack/`, then `docker compose restart slack-mcp` |

## Rollback

To revert to per-session container spawning (project-level `.mcp.json`):

```bash
claude mcp remove --scope user slack
claude mcp remove --scope user grafana
docker compose down
```

Project-level `.mcp.json` entries resume as fallback.

## Architecture

- **Slack**: SSE on `127.0.0.1:3001/sse`, bearer token auth via `SLACK_MCP_API_KEY`
- **Grafana**: SSE on `127.0.0.1:3002/sse`, loopback-only (no auth required)
- Both bound to `127.0.0.1` only - not exposed to the network
