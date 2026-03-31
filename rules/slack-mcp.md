# Slack MCP Usage

## Default Posture: Read-Only

The Slack MCP runs with personal credentials (xoxc/xoxd tokens), not a bot token. All messages sent appear as the user. Treat Slack as read-only unless explicitly writing to the allowlisted target below.

**Read operations (always safe)**:
- `conversations_history` - read channel messages
- `conversations_replies` - read threads
- `conversations_search_messages` - search messages
- `channels_list` - list channels

**Write operations (restricted)**:
- `conversations_add_message` - post a message. Only allowed to the self-DM target.

## Allowed Write Target

The only permitted write destination is the user's self-DM:
- Channel reference: `@npratt`
- User ID: `U04ATDG53T4`

This is enforced server-side via `SLACK_MCP_ADD_MESSAGE_TOOL` in the MCP server .env, which whitelists only the self-DM channel ID. The server rejects attempts to post anywhere else. Do not attempt to post to channels, threads, or other users' DMs.

## Cache Warm-Up

The Slack MCP server uses a local cache that takes time to populate after startup. When you get errors about an empty cache or cache still warming up:

1. Wait 5-10 seconds and retry the same query
2. If it fails again, try a simpler query first (e.g., `channels_list`) to nudge the cache
3. Do not retry more than 3 times - if the cache is genuinely unavailable, tell the user the Slack MCP may need a restart

Common error patterns:
- "cache is empty" - server just started, needs a moment
- "channel not found" - cache may not have indexed this channel yet, retry
- Timeout errors - the initial cache build can be slow on large workspaces
