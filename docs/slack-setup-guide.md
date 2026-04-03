# Slack Setup Guide

Connect Slack as a conversation source so `muse compose` includes your Slack messages.

## Quick Start

```bash
export MUSE_SLACK_TOKEN=xoxp-...        # your user token
muse add slack                           # activate and sync
muse compose                             # includes Slack conversations
```

## Authentication Options

Muse supports three token types via the `MUSE_SLACK_TOKEN` environment variable.

### Option 1: User Token (xoxp-)

The simplest path. Create a Slack app with a user token:

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create a new app
2. Under **OAuth & Permissions**, add these **User Token Scopes**:
   - `search:read` — find your messages across channels
   - `channels:history` — read public channel messages
   - `groups:history` — read private channel messages
   - `im:history` — read direct messages
   - `mpim:history` — read group DMs
   - `users:read` — resolve user display names
3. Install the app to your workspace
4. Copy the **User OAuth Token** (starts with `xoxp-`)

```bash
export MUSE_SLACK_TOKEN=xoxp-...
```

### Option 2: Client Token + Cookie (xoxc-)

Extract from a browser session. The `xoxc-` token requires a `d` cookie:

```bash
export MUSE_SLACK_TOKEN=xoxc-...
export MUSE_SLACK_COOKIE=xoxd-...       # value of the 'd' cookie
```

To extract these from your browser:
1. Open your Slack workspace in a browser
2. Open DevTools → Application → Cookies → find the `d` cookie
3. Open DevTools → Console → run `window.prompt("token", BootData.api_token)` to get the xoxc token

### Option 3: SAML SSO via Cookie File

For enterprise workspaces with SAML SSO. Point `MUSE_SLACK_TOKEN` at a Netscape-format cookie file:

```bash
export MUSE_SLACK_TOKEN=~/cookies.txt
export MUSE_SLACK_WORKSPACE=mycompany.enterprise.slack.com
```

Muse follows the SAML redirect chain using cookies from the file, authenticates with your IDP, and extracts a `xoxc-` token automatically.

For multiple workspaces, comma-separate:

```bash
export MUSE_SLACK_WORKSPACE=team1.enterprise.slack.com,team2.enterprise.slack.com
```

## Required API Scopes

| Scope | Used By | Purpose |
|-------|---------|---------|
| `search:read` | `search.messages` | Discover channels you were active in |
| `channels:history` | `conversations.history` | Fetch public channel messages |
| `groups:history` | `conversations.history` | Fetch private channel messages |
| `im:history` | `conversations.history` | Fetch direct messages |
| `mpim:history` | `conversations.history` | Fetch group DM messages |
| `users:read` | `users.info` | Resolve user IDs to display names |

## Activating the Source

Slack is an opt-in source. Activate it:

```bash
muse add slack
```

This creates an observation directory and syncs conversations. Future `muse compose` runs include Slack automatically.

To deactivate:

```bash
muse remove slack
```

## Verifying It Works

```bash
muse sources                # should show: slack  active  N conversations  N observations
```

## How It Works

1. **Discovery** — searches for all your messages via `search.messages`
2. **Fetch** — downloads full channel history and thread replies for channels you participated in
3. **Cache** — stores raw data at `~/.muse/cache/slack/` (incremental sync on subsequent runs)
4. **Chunk** — splits long channels into ~20k character conversation chunks
5. **Filter** — excludes bot messages, join/leave events, and URL-only messages

Only channels where you actually posted messages are included.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `MUSE_SLACK_TOKEN is not set` | Env var missing | Set `MUSE_SLACK_TOKEN` |
| `slack API error: invalid_auth` | Token expired or invalid | Get a fresh token |
| `slack API error: missing_scope` | Token lacks required scopes | Add scopes listed above |
| `MUSE_SLACK_WORKSPACE not set` | Using cookie file without workspace | Set `MUSE_SLACK_WORKSPACE` |
| `no cookies found in <path>` | Cookie file empty or wrong format | Ensure Netscape cookie format |
| `SAML flow completed but no xoxc token` | SSO cookies expired | Re-export cookies from browser |
| `0 conversations` after sync | Token works but no messages found | Verify the token owner has sent messages |
