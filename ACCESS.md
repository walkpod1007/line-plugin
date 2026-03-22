# LINE — Access & Delivery

A LINE bot added to a chat or searched by username is potentially reachable by anyone. Without a gate, any LINE user who adds your bot would flow straight into your assistant session. The access model described here decides who gets through.

By default, a message from an unknown user triggers **pairing**: the bot replies with a 6-character code and drops the message. You run `/line:access pair <code>` from your assistant session to approve them. Once approved, their messages pass through.

All state lives in `~/.claude/channels/line/access.json`. The `/line:access` skill commands edit this file; the server re-reads it on every inbound message, so changes take effect without a restart. Set `LINE_ACCESS_MODE=static` to pin config to what was on disk at boot (pairing is unavailable in static mode).

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| User ID format | `U` + 32 hex chars (e.g. `U4af4980dc9d2b...`) |
| Group ID format | `C` + 32 hex chars |
| Room ID format | `R` + 32 hex chars |
| Config file | `~/.claude/channels/line/access.json` |

## DM policies

`dmPolicy` controls how DMs from users not on the allowlist are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/line:access pair <code>`. |
| `allowlist` | Drop silently. No reply. |
| `disabled` | Drop everything, including allowlisted users and groups. |

```
/line:access policy allowlist
```

## User IDs

LINE identifies users by opaque string IDs starting with `U`. Unlike Telegram, LINE has no bot-accessible way to look up a user ID without them first messaging you. Pairing captures the ID automatically — it's the easiest way to add someone.

```
/line:access allow U4af4980dc9d2b...
/line:access remove U4af4980dc9d2b...
```

## Groups

Groups are off by default. Opt each one in individually. To find a group's ID, add the bot to the group and send a message — the pairing code flow will display the group/room ID in the server logs (or use `/line:access` to see recently seen IDs in pending entries).

```
/line:access group add Cxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Groups support a member allowlist. Empty = any group member can trigger the bot.

```
/line:access group add Cxxx --allow U4af4980...,Ubcd1234...
```

## Delivery

Configure outbound behavior with `/line:access set <key> <value>`.

**`textChunkLimit`** sets the split threshold. LINE rejects messages over 5000 characters.

**`chunkMode`** chooses the split strategy: `length` cuts exactly at the limit; `newline` prefers paragraph boundaries (default).

```
/line:access set chunkMode length
/line:access set textChunkLimit 2000
```

## Skill reference

| Command | Effect |
| --- | --- |
| `/line:access` | Print current state: policy, allowlist, pending pairings, enabled groups. |
| `/line:access pair a4f91c` | Approve pairing code `a4f91c`. Adds the user to `allowFrom` and sends a confirmation push message. |
| `/line:access deny a4f91c` | Discard a pending code. The user is not notified. |
| `/line:access allow U4af...` | Add a user ID directly without pairing. |
| `/line:access remove U4af...` | Remove from the allowlist. |
| `/line:access policy allowlist` | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`. |
| `/line:access group add Cxxx` | Enable a group or room. Optional: `--allow id1,id2` to restrict members. |
| `/line:access group rm Cxxx` | Disable a group. |
| `/line:access set textChunkLimit 5000` | Set a config key: `textChunkLimit`, `chunkMode`. |

## Config file

`~/.claude/channels/line/access.json`. Absent file = `pairing` policy with empty lists.

```jsonc
{
  // Handling for DMs from users not in allowFrom.
  "dmPolicy": "pairing",

  // LINE user IDs allowed to DM.
  "allowFrom": ["U4af4980dc9d2b..."],

  // Groups/rooms the bot is active in. Empty object = DM-only.
  "groups": {
    "Cxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx": {
      // Restrict triggers to these members. Empty = any member.
      "allowFrom": [],
      // Optional: bot display name for reference.
      "botName": "MyBot"
    }
  },

  // Max chars per outbound message. LINE rejects > 5000.
  "textChunkLimit": 5000,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "newline"
}
```
