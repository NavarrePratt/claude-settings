# Centralized MCP Servers

Local MCP server management via docker-compose for servers that need
persistent local containers. Currently only Slack MCP runs locally;
Grafana/observability is served by a remote MCP server.

## Quick Start

```bash
cd ~/.claude/mcp-servers
cp .env.example .env   # Fill in Slack tokens
docker compose up -d
```

## Management

```bash
docker compose up -d                          # Start
docker compose down                           # Stop
docker compose ps                             # Check status
docker compose logs -f slack-mcp              # Tail logs
docker compose pull && docker compose up -d   # Update images
```

## Token Rotation

1. Edit `.env` with new token values
2. `docker compose restart slack-mcp`
3. For `SLACK_MCP_API_KEY` changes: also update the bearer token in Claude MCP
   config (`claude mcp remove --scope user slack` then re-add with new header)

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Connection refused | `docker compose ps` - verify container is running and port is bound |
| Auth errors (401) | Verify `SLACK_MCP_API_KEY` in `.env` matches the `Authorization` header in Claude MCP config |
| Blank responses | `docker compose logs -f slack-mcp` - look for upstream API errors |
| Cache stale | Remove contents of `data/slack/`, then `docker compose restart slack-mcp` |

## Rollback

To revert Slack to per-session container spawning:

```bash
claude mcp remove --scope user slack
docker compose down
```

Project-level `.mcp.json` entries resume as fallback.

## Architecture

- **Slack**: local SSE on `127.0.0.1:3001/sse`, bearer token auth via `SLACK_MCP_API_KEY`, bound to loopback only
- **Grafana/Observability**: remote MCP server (no local container, requires VPN)
