---
name: configure
description: Set up the LINE channel — save the Channel Access Token and Channel Secret. Use when the user pastes LINE credentials, asks to configure LINE, asks "how do I set this up", or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /line:configure — LINE Channel Setup

Writes LINE credentials to `~/.claude/channels/line/.env` and orients the
user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Credentials** — check `~/.claude/channels/line/.env` for
   `LINE_CHANNEL_ACCESS_TOKEN` and `LINE_CHANNEL_SECRET`. Show set/not-set;
   if set, show first 10 chars masked (e.g. `ABCdef1234...`).

2. **Webhook port** — show `LINE_WEBHOOK_PORT` (default: 3000).

3. **Access** — read `~/.claude/channels/line/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed users: count and list IDs
   - Pending pairings: count, with codes and user IDs if any

4. **What next** — end with a concrete next step based on state:
   - No credentials → *"Run `/line:configure token <token>` then
     `/line:configure secret <secret>` with credentials from LINE Developers
     Console."*
   - Credentials set, nobody paired → *"Message your LINE bot. It replies
     with a code; approve with `/line:access pair <code>`."*
   - Credentials set, someone paired → *"Ready. Message your LINE bot to
     reach the assistant."*
   - Always remind: LINE requires a public HTTPS webhook — use ngrok or
     Cloudflare Tunnel to expose port 3000 (or `LINE_WEBHOOK_PORT`).

**Push toward lockdown — always.** Once IDs are captured via pairing,
switch to `allowlist` policy. Offer to run `/line:access policy allowlist`.

### `token <value>` — save Channel Access Token

1. Treat the argument as the token (trim whitespace).
2. `mkdir -p ~/.claude/channels/line`
3. Read existing `.env` if present; update/add the `LINE_CHANNEL_ACCESS_TOKEN=` line,
   preserve other keys. Write back, no quotes around the value.
4. Confirm, then show the no-args status.

### `secret <value>` — save Channel Secret

Same as above but for `LINE_CHANNEL_SECRET=`.

### `port <number>` — set webhook port

Update/add `LINE_WEBHOOK_PORT=<number>` in `.env`.

### `clear` — remove credentials

Delete `LINE_CHANNEL_ACCESS_TOKEN=` and `LINE_CHANNEL_SECRET=` lines
(or the whole file if those are the only lines).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet.
  Missing file = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/line:access` take effect immediately, no restart needed.
- Remind the user that LINE requires setting the webhook URL in the
  LINE Developers Console → Messaging API → Webhook URL:
  `https://<tunnel-host>/webhook`
- Webhook URL verification in LINE console requires the server to be running
  and the tunnel to be active.
- **Follow event pairing is active**: when a new user adds the bot as a friend,
  they immediately receive a pairing code without needing to send a message first.
  Existing allowlisted users get a "Welcome back" message instead.
