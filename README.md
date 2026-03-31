# line-plugin

LINE MCP server for Claude Code, extended for Life-OS.

Based on [breakingmind/claude-external-plugins-line](https://github.com/breakingmind/claude-external-plugins-line). The original project provides the core LINE webhook → MCP bridge. This fork adds Life-OS–specific context injection and automation layers on top.

---

## Extensions (Life-OS specific)

The following features are added beyond the original project:

| Extension | Description |
| --- | --- |
| **STATE.md / flag.md injection** | On each inbound message, `~/.claude/STATE.md` (人類近況 + 阿普觀察) and `~/.claude/flag.md` (6-channel boost keywords, LRU 80 entries) are read and prepended to the Claude context. |
| **boost_keywords tool** | Semantic deduplication + LRU eviction on the keywords ring buffer. Writes back to `flag.md` after each turn. |
| **turn_protocol** | Reply format rules enforced per turn: length cap, no filler praise, direct answers first. |
| **scene-cut detection** | Detects topic shifts mid-conversation and appends a timestamped entry to `~/.claude/projects/.../daily/YYYY-MM-DD.md`. |
| **Taiwan timezone timestamps** | All log entries and session markers use `Asia/Taipei` (UTC+8). |
| **Morning brief trigger** | First inbound message after 07:00 Asia/Taipei triggers the `morning-brief` skill automatically. |
| **Token tracking** | Each reply footer includes: tokens used this turn / session total / today's total. |
| **/restart command** | User sends `/restart` in LINE → Claude Code process is restarted via `claude-supervisor.sh`. |
| **Zhihu extraction via Chrome cookies** | Fetches Zhihu article content by decrypting Chrome's local cookie store, bypassing login walls. |

---

## Setup

For LINE bot creation, webhook configuration, plugin installation, credential storage, pairing, and access control, see the [original project README](https://github.com/breakingmind/claude-external-plugins-line).

---

## Life-OS specific configuration

### Context injection files

| File | Purpose |
| --- | --- |
| `~/.claude/STATE.md` | Human status + AI observations, updated nightly by Gemini |
| `~/.claude/flag.md` | Boost keywords LRU ring buffer (80 entries, 6 channels) |

Both files are auto-read on each incoming message. No extra setup needed once they exist.

### Morning brief

The `morning-brief` skill must be defined under `~/Documents/Life-OS/skills/morning-brief/SKILL.md`. The LINE plugin checks the current hour on message receipt and fires the skill if the hour >= 7 and no brief has been sent today (tracked in `~/.claude/channels/line/.morning_brief_date`).

### Token tracking

Token counts are pulled from Claude Code's internal session state and appended as a footer:

```
─
tokens: 1,234 this turn | 45,678 session | 123,456 today
```

### /restart command

When the LINE message body equals `/restart` exactly:
1. The plugin replies "重啟中..." via LINE.
2. It calls `bash ~/Documents/Life-OS/scripts/claude-supervisor.sh restart`.
3. The new Claude process inherits the same channel config.

### Zhihu extraction

Used by the `capture` skill when a `zhihu.com` URL is detected. The plugin locates Chrome's `Cookies` SQLite file, decrypts the `z_c0` cookie using the macOS Keychain master key, and injects it into the fetch request. No credentials are stored separately.

---

## 繁體中文說明

### 擴充功能總覽

本 fork 基於 [breakingmind/claude-external-plugins-line](https://github.com/breakingmind/claude-external-plugins-line) 原專案，在核心 LINE webhook → MCP 橋接基礎上，加入以下 Life-OS 專屬功能：

- **STATE.md / flag.md 注入**：每則收到的訊息，自動將人類近況與關鍵字清單注入 Claude 上下文。
- **boost_keywords 工具**：語意去重 + LRU 淘汰機制，維護 80 條關鍵字緩衝。
- **turn_protocol 回覆規範**：每輪強制執行回覆格式規則（長度上限、禁止填充讚美、直接回應）。
- **scene-cut 偵測**：偵測話題切換，寫入當日對話日誌。
- **台灣時區時間戳記**：所有記錄使用 Asia/Taipei（UTC+8）。
- **晨報觸發**：07:00 後第一則訊息自動觸發 `morning-brief` 技能。
- **token 追蹤**：每則回覆顯示本輪 / Session / 今日 token 用量。
- **/restart 指令**：LINE 傳送 `/restart` → 透過 `claude-supervisor.sh` 重啟 Claude Code。
- **知乎擷取（Chrome cookie 解密）**：自動解密 Chrome 本地 cookie 存取知乎內容，無需手動登入。

### 基本設定

LINE Bot 建立、webhook 設定、外掛安裝、憑證儲存、配對與存取控制，請參考[原專案 README](https://github.com/breakingmind/claude-external-plugins-line)。
