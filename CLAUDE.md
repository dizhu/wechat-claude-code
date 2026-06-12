# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

A bridge that lets you chat with the local `claude` CLI from WeChat. A Node/TypeScript
daemon long-polls Tencent's ilink Bot API, forwards WeChat messages to a spawned
`claude` process, and streams replies back. **This is a security-hardened, multi-user
fork** of `Wechat-ggGitHub/wechat-claude-code`.

> **Before changing anything, read [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md).** It lists every
> local hardening/feature change and *why*. Do not silently revert those — several guard
> against real security or concurrency bugs. If a change touches one, keep the protection.

## Commands

- `npm run build` — compile TS → `dist/` (tsc). Run after every source edit.
- `npm run daemon -- start|stop|restart|status|logs` — manage the launchd/systemd daemon.
  Use `restart` after `npm run build` to load changes.
- `npm run setup` — QR-bind a WeChat account (each run binds one more bot).
- `npm test` — **no real tests exist** (`dist/tests/*` is empty; the script exits 0 silently).
  Verify changes by `build` + `restart` + reading logs, not by `npm test`.

There is no lint step and no test suite. The compiler (`strict: true`) is the only gate.

## Architecture

Message flow: `WeChat → ilink Bot API → daemon (long-poll) → spawn claude → stream back`.

- `src/main.ts` — entrypoint. `runDaemon()` loads **all** accounts and starts one
  `createBotMonitor(account)` per bot; `handleMessage` / `sendToClaude` do the per-message work.
- `src/wechat/` — `monitor.ts` (long-poll loop, per-bot), `api.ts` (WeChatApi + `isTrustedWechatUrl`),
  `send.ts`, `media.ts`, `upload.ts`, `accounts.ts` (`loadAllAccounts`), `sync-buf.ts` (per-bot cursor),
  `login.ts`, `crypto.ts`.
- `src/claude/provider.ts` — spawns `claude`, parses NDJSON stdout, handles abort/timeout.
- `src/session.ts` / `src/store.ts` — per-user session persistence (JSON files).
- `src/commands/` — slash-command routing (`/clear`, `/stop`, `/prompt`, `/cwd`, …).
- `src/config.ts` — global config (`~/.wechat-claude-code/config.json`).

Data lives in `~/.wechat-claude-code/`: `accounts/` (one `<accountId>.json` per bot),
`sessions/` (`<accountId>__<user>.json`), `sync/` (one cursor per bot), `config.json`, `logs/`.

## Multi-user / multi-bot model

ilink bots are **1:1** — one bot per WeChat user; you cannot add others to one bot. So
"multiple members" = **one daemon serving N bots**, each member binding their own. The daemon
runs one monitor per bot; everything is keyed per user inside each bot.

## Invariants — do not break these

- **Per-bot sync cursor** (`sync-buf.ts`): the long-poll cursor is bot-specific (encodes the bot
  id). It MUST be keyed by `accountId`. A single shared cursor file makes bots clobber each other.
- **Session keys** (`main.ts` `sessionKeyFor`): `accountId__` + (`r_<userId>` if charset-safe, else
  `b_<base64url>`). The `r_`/`b_` prefixes keep the two namespaces disjoint — keep them or two users
  can collide onto one session.
- **`drainQueue` ordering** (`main.ts`): set `rt.processing = false` BEFORE the queue re-check in
  `finally`. The re-enter guard at the top is what prevents double-processing.
- **`/stop` abort** (`provider.ts` + `main.ts`): abort resolves with `aborted: true` (not a throw);
  `sendToClaude` must early-return on `result.aborted` (incl. after the resume retry) so partial
  output isn't sent and the aborted session id isn't persisted.
- **Per-bot crash isolation** (`runResilient` in `main.ts`): one bot's monitor crashing must not
  bring down the others.
- **Fail-closed startup**: accounts missing `userId`, or with an invalid `accountId`, are skipped —
  never run a bot wide open.
- **Security guards**: `media.ts` basename+containment (path traversal), `isTrustedWechatUrl` on
  every server-supplied URL, SIGTERM→SIGKILL escalation in `provider.ts`. See `LOCAL_PATCHES.md`.

## Security context

`claude` runs with `--dangerously-skip-permissions` (full command execution) — an accepted
decision. **Messaging a bot ≈ running arbitrary commands on the host.** Sender authorization
(owner-only per bot, by `from_user_id`) is therefore the load-bearing defense; don't weaken it.
Never log bot tokens or AES keys (the logger redacts `*token`/`*secret`/`aes_key`; don't bypass it).

## Conventions

- TypeScript ESM, `.js` import suffixes, `strict: true`, no extra deps beyond `qrcode`/`qrcode-terminal`.
- Match the existing terse style; comments explain *why* (constraints/invariants), not *what*.
- After editing source: `npm run build` then `npm run daemon -- restart`, then check
  `~/.wechat-claude-code/logs/bridge-<date>.log`.
- Keep `LOCAL_PATCHES.md` and `LOCAL_PATCHES.patch` (`git diff -- src/ > LOCAL_PATCHES.patch`)
  in sync when you change hardened code, so upstream merges stay auditable.
