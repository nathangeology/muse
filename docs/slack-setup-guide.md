# Slack Setup Guide

Connect Slack as a conversation source so `muse compose` includes your Slack messages.

## Quick Start

```bash
# 1. Set credentials
export MUSE_SLACK_TOKEN=xoxp-...          # Slack user token
export MUSE_SLACK_WORKSPACE=mycompany.enterprise.slack.com  # only needed for SSO

# 2. Activate the source
muse add slack

# 3. Compose (Slack is now included automatically)
muse compose
```

## Authentication

Muse reads `MUSE_SLACK_TOKEN`. The value determines the auth method:

### Option A: User OAuth Token (simplest)

Use a `xoxp-` token from a Slack app you control.

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create an app (or use an existing one).
2. Under **OAuth & Permissions**, add the scopes listed below.
3. Install the app to your workspace.
4. Copy the **User OAuth Token** (`xoxp-...`).

```bash
export MUSE_SLACK_TOKEN=xoxp-your-token-here
```

No other env vars needed — the token authenticates directly.

### Option B: SAML SSO via Cookie File

For enterprise workspaces that use SAML SSO (e.g. Okta, Azure AD), point
`MUSE_SLACK_TOKEN` at a Netscape-format cookie file exported from your browser.

1. Log into your Slack workspace in a browser.
2. Export cookies to a file using a browser extension (e.g. "Get cookies.txt LOCALLY").
3. Set both env vars:

```bash
export MUSE_SLACK_TOKEN=~/path/to/cookies.txt
export MUSE_SLACK_WORKSPACE=mycompany.enterprise.slack.com
```

Muse detects the file path (starts with `/` or `~/`), loads the cookies, follows
the SAML redirect chain, and extracts a `xoxc-` session token automatically.

For multiple workspaces, comma-separate them:

```bash
export MUSE_SLACK_WORKSPACE=team1.enterprise.slack.com,team2.enterprise.slack.com
```

### Option C: Manual xoxc Token + Cookie

If you have a `xoxc-` token (e.g. extracted from browser dev tools), pair it with
the `d` cookie:

```bash
export MUSE_SLACK_TOKEN=xoxc-your-token-here
export MUSE_SLACK_COOKIE=your-d-cookie-value
```

`xoxc-` tokens require the `d` cookie on every request — without `MUSE_SLACK_COOKIE`
API calls will fail with `invalid_auth`.

## Required Slack API Scopes

The provider uses these API methods:

| Method | Scope Required | Purpose |
|--------|---------------|---------|
| `auth.test` | (no extra scope) | Verify token, get user/team ID |
| `search.messages` | `search:read` | Find channels you were active in |
| `conversations.history` | `channels:history`, `groups:history`, `im:history`, `mpim:history` | Fetch channel messages |
| `conversations.replies` | `channels:history`, `groups:history`, `im:history`, `mpim:history` | Fetch thread replies |
| `users.info` | `users:read` | Resolve user IDs to display names |

**Minimum scope set for a `xoxp-` token:**
- `search:read`
- `channels:history`
- `groups:history`
- `im:history`
- `mpim:history`
- `users:read`

SSO-derived tokens (`xoxc-`) inherit the scopes of the authenticated user session,
which typically includes all of the above.

## Activating the Source

Slack is an opt-in source. Activate it with:

```bash
muse add slack
```

This creates the observation directory and runs an initial sync. After activation,
`muse compose` includes Slack automatically on every run.

Check status:

```bash
muse sources
```

To deactivate:

```bash
muse remove slack
```

## How It Works

1. **Discovery** — Searches for all messages you sent (`from:<@your_id>`) to find
   channels you were active in.
2. **Fetch** — For each active channel, fetches the message history and thread replies
   in your activity time range.
3. **Assembly** — Flattens threads into a chronological timeline per channel, filters
   noise (bot messages, join/leave events, bare URLs), and chunks into ~20k-character
   conversations.
4. **Caching** — Results are cached at `~/.muse/cache/slack/`. Subsequent runs only
   fetch messages newer than the last sync.

## Verifying It Works

```bash
# Check the source is active and has conversations
muse sources

# Run compose and watch Slack sync progress
muse compose
```

You should see output like:

```
authenticated via SSO (~/cookies.txt → mycompany.enterprise.slack.com)
mycompany: 12 channels, 847 messages
```

## Troubleshooting

### `MUSE_SLACK_TOKEN is not set, skipping Slack source`

The env var is empty or unset. Export it in your shell profile.

### `MUSE_SLACK_WORKSPACE not set`

Required when `MUSE_SLACK_TOKEN` points to a cookie file. Set it to your
workspace domain (e.g. `mycompany.enterprise.slack.com`).

### `slack API error: invalid_auth`

- **xoxp token**: Token may be revoked or expired. Generate a new one.
- **xoxc token**: Missing or stale `MUSE_SLACK_COOKIE`. Re-export from browser.
- **SSO**: Cookie file may be expired. Re-export cookies after logging in.

### `slack API error: missing_scope`

Your token lacks a required scope. See the scopes table above. For `xoxp-` tokens,
add the missing scope in your Slack app's OAuth settings and reinstall.

### `SAML flow completed but no xoxc token found in response`

The SSO redirect chain completed but Slack didn't return a token. This usually means:
- The cookie file is stale (re-export after a fresh browser login)
- The workspace URL is wrong (check `MUSE_SLACK_WORKSPACE`)
- Your IDP session expired

### `no cookies found in <path>`

The cookie file exists but contains no parseable cookies. Ensure it's in
Netscape/Mozilla cookie format (tab-separated, one cookie per line).

### Rate limiting (slow sync)

Muse respects Slack's rate limits with adaptive pacing:
- `search.messages`: ~0.5 req/s (Tier 2)
- `conversations.history` / `conversations.replies`: ~2 req/s (Tier 3)

First sync of a large workspace may take several minutes. Subsequent syncs are
incremental and much faster.

### Clearing the cache

To force a full re-sync:

```bash
rm -rf ~/.muse/cache/slack/
muse compose
```
