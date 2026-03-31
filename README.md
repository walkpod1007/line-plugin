# line-plugin

> LINE Messaging API MCP server for Claude Code, with Life-OS extensions.
> Inspired by [breakingmind/claude-external-plugins-line](https://github.com/breakingmind/claude-external-plugins-line).

---

## Features

| Extension | Description |
| --- | --- |
| **STATE.md / flag.md injection** | Each inbound message prepends `~/.claude/STATE.md` (人類近況 + 阿普觀察) and `~/.claude/flag.md` (boost keywords, LRU 80 entries) into Claude context |
| **boost_keywords tool** | Semantic deduplication + LRU eviction on the keywords ring buffer; writes back to `flag.md` after each turn |
| **turn_protocol** | Enforces reply format per turn: length cap, no filler praise, direct answer first |
| **scene-cut detection** | Detects topic shifts mid-conversation and appends a timestamped entry to the daily log file |
| **Taiwan timezone timestamps** | All log entries and session markers use `Asia/Taipei` (UTC+8) |
| **Morning brief trigger** | First message after 07:00 Asia/Taipei automatically fires the `morning-brief` skill |
| **Token tracking** | Each reply footer shows: tokens this turn / session total / today's total |
| **/restart command** | Send `/restart` in LINE → restarts the Claude Code process via `claude-supervisor.sh` |
| **Zhihu extraction via Chrome cookies** | Decrypts Chrome's local cookie store to fetch Zhihu articles without manual login |

---

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [Bun](https://bun.sh) runtime (`brew install bun`)
- LINE Messaging API account (free tier works)
- A publicly reachable HTTPS endpoint — [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) is the recommended zero-config option

---

## Quick Start

### 1. Create LINE Bot

1. Go to [LINE Developers Console](https://developers.line.biz/) and create a new provider + Messaging API channel.
2. Under **Messaging API** tab: note the **Channel Access Token** (issue one if needed) and **Channel Secret**.
3. Disable auto-reply messages (Messaging API → Auto-reply messages → Disabled) so Claude handles all replies.

### 2. Install

```bash
git clone https://github.com/walkpod1007/line-plugin.git
cd line-plugin
bun install
```

### 3. Configure

```bash
mkdir -p ~/.claude/channels/line
cat > ~/.claude/channels/line/.env << 'EOF'
LINE_CHANNEL_ACCESS_TOKEN=your_token
LINE_CHANNEL_SECRET=your_secret
LINE_WEBHOOK_PORT=3000
EOF
```

### 4. Set Webhook URL

Start a Cloudflare Tunnel pointing at the local port:

```bash
cloudflared tunnel --url http://localhost:3000
```

Copy the generated `https://*.trycloudflare.com` URL and paste it into LINE Developers Console → **Webhook URL** (append `/webhook`), then click **Verify**.

### 5. Connect to Claude Code

Add the following to your `.mcp.json` (or `~/.claude/mcp.json` for global scope):

```json
{
  "mcpServers": {
    "line-plugin": {
      "command": "/opt/homebrew/bin/bun",
      "args": ["/path/to/line-plugin/server.ts"]
    }
  }
}
```

Replace `/path/to/line-plugin` with the actual clone path, then restart Claude Code.

---

## Life-OS Extensions

Two files are auto-read on every incoming message — no extra setup once they exist:

| File | Purpose |
| --- | --- |
| `~/.claude/STATE.md` | Human status + AI observations, updated nightly by Gemini |
| `~/.claude/flag.md` | Boost keywords LRU ring buffer (80 entries, 6 channels) |

**Morning brief**: The `morning-brief` skill must be defined at `~/Documents/Life-OS/skills/morning-brief/SKILL.md`. Today's brief status is tracked in `~/.claude/channels/line/.morning_brief_date` to avoid double-firing.

**Token footer** format appended to every reply:

```
─
tokens: 1,234 this turn | 45,678 session | 123,456 today
```

---

## Commands

`/restart` — Replies "重啟中..." then calls `bash ~/Documents/Life-OS/scripts/claude-supervisor.sh restart` to reboot the Claude Code process.

---

## Credits

Based on [breakingmind/claude-external-plugins-line](https://github.com/breakingmind/claude-external-plugins-line), which provides the core LINE webhook → MCP bridge. This fork adds Life-OS–specific context injection, automation layers, and the extensions listed above.
