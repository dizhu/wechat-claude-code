# WeChat Claude Code Bridge

<p align="center">
  <strong>Chat with Claude Code in WeChat, just like texting a friend</strong>
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License: MIT"></a>
  <a href="README.md"><img src="https://img.shields.io/badge/Lang-中文-lightgrey?style=flat-square" alt="中文"></a>
</p>

Scan a QR code to bind your WeChat, and a new "friend" appears in your contacts. Send it a message — it gets forwarded to Claude Code running on your computer, and the reply streams back to WeChat in real time. Supports text, images, voice, and files.

> **This repo is a security-hardened, multi-user fork of [Wechat-ggGitHub/wechat-claude-code](https://github.com/Wechat-ggGitHub/wechat-claude-code).**
> It adds 20+ security/correctness hardening fixes on top of upstream and extends the single-user bridge so that **one daemon serves multiple bots / multiple members, each with a fully isolated session.**
> The full list of local changes and rationale is in [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md). **Read the [⚠️ Security notes](#-security-notes) before using.**

## Highlights

| | |
|---|---|
| **Scan and go** | No account signup, no server deployment. Scan a QR code and you're done in a minute. Credentials stay on your machine. |
| **Multi-user / team** | One daemon serves multiple members; each binds their own bot, with isolated sessions, context, and working directories. |
| **Clean messages** | Only key info gets pushed — progress, results, key decisions. Tool-call noise is filtered out automatically. |
| **"Typing..." indicator** | WeChat shows a typing indicator while Claude is working. |
| **Consistent experience** | Mobile and desktop Claude Code behave identically — same orchestration, same output. |
| **Two-way files** | Send images, Word docs, PDFs for Claude to analyze; files Claude generates get pushed straight to WeChat. |
| **Timeout reassurance** | Task taking longer than 5 minutes? You'll get an automatic "still working" message. |
| **Hardened** | Sender auth, path-traversal protection, upload-URL allowlist, zombie-process cleanup, fail-closed startup, and more — see `LOCAL_PATCHES.md`. |

## Install

```bash
git clone https://github.com/dizhu/wechat-claude-code.git ~/.claude/skills/wechat-claude-code
cd ~/.claude/skills/wechat-claude-code && npm install
```

> Requires Node.js >= 18, macOS or Linux, a personal WeChat account, and an installed + authenticated [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI.
> `npm install` auto-runs `npm run build` to compile the TypeScript.

## Quick start (single user)

### 1. Bind via QR

```bash
cd ~/.claude/skills/wechat-claude-code
npm run setup
```

A QR code pops up — scan and confirm it in WeChat.

### 2. Start the service

```bash
npm run daemon -- start
```

On macOS this registers a launchd service (auto-start on boot, auto-restart on crash). On startup it prints the number of bots served, e.g. `已启动 (1 个 bot: xxx@im.bot)`.

### 3. Start chatting

Open WeChat and message the new "friend".

### Manage the service

```bash
npm run daemon -- status   # status
npm run daemon -- stop     # stop
npm run daemon -- restart  # restart (after updating code or adding a member)
npm run daemon -- logs     # logs
```

## Multi-user / team use

An ilink bot is **1:1** — each bot binds exactly one WeChat account, and you cannot share its contact card to let others join the same bot.
So the correct way to support a team is: **each member scans once and binds their own dedicated bot, all served by the same daemon.**
Each member's session context, chat history, and working directory are fully isolated; different members' requests run concurrently without blocking each other.

**Add a member:**

1. Generate a binding QR on the host:
   ```bash
   npm run setup
   ```
2. Send the QR image (`~/.wechat-claude-code/qrcode.png`) to the member and have them **scan and confirm it with their own phone's WeChat**.
   The bot appears in **their** WeChat; credentials are saved to `~/.wechat-claude-code/accounts/<new-id>.json` on the host.
3. Restart so the daemon picks up the new bot:
   ```bash
   npm run daemon -- restart
   ```
   The startup log shows the total bot count, e.g. `已启动 (2 个 bot: ...)`.

> Each member gets a default working directory `<config.workingDirectory>/<short-user-id>`, created automatically; members can switch with `/cwd`.

## WeChat commands

Send these directly in the chat:

| Command | Description |
|---------|-------------|
| `/help` | Show help |
| `/clear` | Clear the current session (keeps working dir / model / prompt) |
| `/stop` | Stop the current task and clear queued messages |
| `/model <name>` | Switch Claude model |
| `/prompt <text>` | Set a system prompt (**only affects you**) |
| `/cwd <path>` | Switch your own working directory |
| `/skills` | List installed skills |
| `/status` | Show session status |
| `/history [n]` | Show recent conversation |
| `/compact` | Compact context, start a new CLI session |
| `/reset` | Full reset (including working dir) |
| `/undo [n]` | Undo recent turns |
| `/<skill> [args]` | Invoke any installed skill |

## How it works

```
Member A WeChat ─┐
Member B WeChat ─┼─→ ilink Bot API ─→ Node.js daemon ─→ Claude Code CLI (local, per-member isolated sessions)
Member C WeChat ─┘                       (one polling loop + independent cursor per bot)
```

The daemon runs one independent long-poll loop per bound bot (each with its own cursor, never overwriting each other), routes incoming messages to an isolated session by sender, forwards to the local `claude` CLI, and streams replies back to WeChat. Everything runs on your own machine.

## ⚠️ Security notes

**Understand this: the tool lets WeChat messages execute arbitrary commands on the host with full permissions.**

- **Full-permission execution**: `claude` runs with `--dangerously-skip-permissions` — no confirmation prompts (WeChat can't answer prompts).
  **Being able to message a bot ≈ being able to run any command on the host** — read/write files, run shell, network access.
- **Shared machine, shared credentials**: every member's Claude runs under the **same machine and OS user**. Sessions are isolated, but the **machine is not** — any member's Claude can read this machine's files, SSH keys, and tokens.
  👉 **Strongly prefer running the daemon on a dedicated machine without sensitive credentials, not your main dev box.**
- **Credential safety**: bot tokens live in `~/.wechat-claude-code/accounts/` (mode `0600`). **Don't share any bot contact with unrelated people; don't leak the accounts dir.**
- **Account-ban risk**: this is an unofficial bridge on personal WeChat; multiple bots and sustained traffic increase the risk of triggering WeChat's anti-abuse controls.
- **Authorization**: by default only the bound owner can drive their own bot; a bot whose account lacks a bound user is fail-closed (skipped) at startup.

See [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md) for hardening details.

## Prerequisites

- Node.js >= 18
- macOS or Linux
- A personal WeChat account (one per member)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed and authenticated

> **Tip:** Claude Code supports third-party API providers (OpenRouter, AWS Bedrock, etc.) via `ANTHROPIC_BASE_URL` and `ANTHROPIC_API_KEY`.
> Note: when multiple members share one daemon, all usage bills to the single API key configured on the host.

## Data directory

Everything is stored under `~/.wechat-claude-code/`:

```
~/.wechat-claude-code/
├── accounts/       # Per-bot WeChat credentials (one <accountId>.json each, 0600)
├── config.json     # Global config (working dir, model, system prompt, etc.)
├── sessions/       # Session data (keyed <accountId>__<user>, isolated per member)
├── sync/           # Long-poll cursors (one <accountId>.json per bot, never overwritten)
└── logs/           # Logs (daily rotation, 30-day retention)
```

## Credits

Forked from [Wechat-ggGitHub/wechat-claude-code](https://github.com/Wechat-ggGitHub/wechat-claude-code) under the MIT license.
This fork adds multi-user / multi-bot support and a series of security/correctness hardening fixes — see [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md).

## License

[MIT](LICENSE)
