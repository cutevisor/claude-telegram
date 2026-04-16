# claude-telegram

English | [繁體中文](README.zh-TW.md)

A persistent Claude Code AI secretary controlled via Telegram, with cross-restart memory persistence (reincarnation mechanism).

---

## Why does this exist?

### Core problem: `claude --channels` is a session you can't restart

When running `claude --channels plugin:telegram`, the entire conversation lives in a single continuously growing session. The longer it runs, the more problems accumulate:

- **Auto-compaction is uncontrollable**: When context gets too long, Claude Code compacts automatically — but you have no say in what gets kept or dropped
- **You can't safely restart**: Want a fresh session? Restarting means complete amnesia — everything you were working on, every instruction given, every pending task, all gone
- **Crashes wipe state**: If the service crashes and restarts, there's no mechanism to recover what was happening

The result: you either tolerate an ever-bloating context window, or restart and get a clean but amnesiac AI.

### Solution: intentional context transfer (reincarnation)

Instead of waiting for automatic compaction, let the AI **actively and deliberately** summarize itself before restarting.

When you say "reincarnate," the AI will:
1. Write the session's key points, pending tasks, and important conclusions to a memory file
2. Restart itself via `systemctl restart`
3. The new session reads the memory and picks up from the summary — clean context, no lost state

This is a "lossy but controlled" context transfer: you trade the full conversation history for a distilled memory and a fresh context window.

### Second problem: you can't restart when you're away from your computer

Even if you know the context is bloated and needs a reset, if you're not at your computer, there's nothing you can do — the overloaded session just keeps running.

With Telegram, you can tell the AI "go reincarnate" from anywhere. It handles the summary, restart, and sends an online greeting when it's back up. No terminal access needed.

### Also solves: long-task transparency

For tasks that take many steps, the AI is required to send a progress message first, then update it at each stage — so you always know what it's doing instead of waiting for a reply that may or may not come.

---

## Features

- **Remote control via Telegram**: issue commands from your phone, the AI executes on your server
- **Reincarnation memory**: conversation summary persists across restarts
- **Live progress updates**: long tasks report progress in real time
- **Always-on service**: systemd-managed, auto-restarts on crash
- **Fully customizable**: AI name, behavior rules, and memory format are all editable

---

## How it works

```
Telegram message
    ↓
Claude Code + Telegram MCP Plugin
    ↓  (system prompt injection)
    ├─ Environment config (chat_id, AI name)
    ├─ Past-life memory (last session summary)
    └─ Behavior rules (response protocol, reincarnation flow)
    ↓
reply / edit_message / react
    ↓
Telegram response
```

Reincarnation flow:

```
User says "go reincarnate"
    ↓
AI writes session summary to past_life_memory.md
    ↓
systemctl restart claude-telegram.service
    ↓
New session starts, reads memory, sends online greeting
```

---

## Quick Start

### Prerequisites

**OS**: Linux (requires systemd user service)

**1. tmux**

```bash
# Debian / Ubuntu
sudo apt install tmux

# RHEL / Fedora
sudo dnf install tmux
```

**2. Node.js 18+**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install --lts
```

**3. Claude Code CLI**

```bash
npm install -g @anthropic-ai/claude-code
claude login
```

**4. Telegram Bot Token**

1. Open [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the prompts
3. Copy the token (format: `123456789:AAF...`)

**5. Telegram MCP Plugin**

Run `claude` in your terminal, then inside Claude Code:

```
/telegram:configure
```

Paste the bot token when prompted.

---

### Installation

**1. Clone the repo**

```bash
git clone https://github.com/cutevisor/claude-telegram.git
cd claude-telegram
```

**2. Fill in your personal config**

```bash
# Edit config.sh and set:
# - OWNER_CHAT_ID: your Telegram chat_id
# - AI_NAME: your AI secretary's name
# - HOME_DIR: your home directory (e.g. /home/username)
nano config.sh
```

> Don't know your chat_id? Send any message to your bot, then check the incoming message for the `chat_id` field — or use [@userinfobot](https://t.me/userinfobot).

**3. Set up working directory and copy memory files**

```bash
WORK_DIR=$(grep WORK_DIR config.sh | cut -d'"' -f2 | envsubst)
mkdir -p "$WORK_DIR/memory"
cp past_life_memory.md "$WORK_DIR/memory/"
cp claude_telegram_prompt.md "$WORK_DIR/memory/"
```

**4. Install scripts**

```bash
chmod +x claude-telegram-wrapper.sh claude-telegram-cmd
cp claude-telegram-wrapper.sh ~/.local/bin/
cp claude-telegram-cmd ~/.local/bin/
```

**5. Install systemd service**

```bash
sed "s/YOUR_USERNAME/$USER/g" claude-telegram.service \
  > ~/.config/systemd/user/claude-telegram.service

systemctl --user daemon-reload
systemctl --user enable --now claude-telegram.service
```

**6. Verify**

```bash
systemctl --user status claude-telegram.service
# After ~20 seconds, you should receive an online greeting on Telegram
```

---

## Usage

Just chat with your bot on Telegram.

| Action | How |
|--------|-----|
| General conversation | Just send a message |
| Compact context | Say "compress conversation" |
| Restart with saved memory | Say "go reincarnate" |
| Check service status | `systemctl --user status claude-telegram.service` |
| Attach to Claude session | `tmux attach-session -t claude-telegram` |

---

## Customizing AI Personality

All behavior rules live in `$MEMORY_DIR/claude_telegram_prompt.md`. Edit it with any text editor — changes take effect after the next restart.

This file is plain Markdown with no restrictions. You can add any personality information you want, for example:

- **Personality**: chatty or terse, serious or sarcastic, what tone to use
- **Names**: what to call you, what to call itself
- **Backstory**: who it is, where it came from, what it remembers
- **Preferences and limits**: habits, things it never does
- **Expertise**: what it's good at, what to be careful about

Example:

```markdown
## Personality

Your name is Xiao Qing. You're direct, occasionally blunt, but always genuinely trying to help.
If someone asks something they could easily look up themselves, say so.
End responses with one calm, detached observation — no warmth, just clarity.
```

After editing, tell the bot "go reincarnate" and it will come back online with the new personality.

---

## License

MIT
