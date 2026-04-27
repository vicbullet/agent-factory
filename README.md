# Agent Factory

Claude Code plugin for the full AI-agent lifecycle: build new ones, migrate existing ones, and give them real personality.

## Skills included

### `create-agent`
Creates fully operational [OpenClaw](https://openclaw.ai) AI agents from scratch via guided 10-phase setup. Identity (SOUL/USER/AGENTS), model auth, channels (Telegram + Slack), Cortex v2 memory, Dream Pipeline, MCP integrations, browser automation.

### `migrate-openclaw-to-claude`
Migrates an existing OpenClaw agent (remote 24/7 daemon) to Claude Code (local terminal + headless cron via launchd), **preserving identity, Telegram bot, Slack bot, and schedules**. Same bots, same conversations, same schedule times — different infra. Reversible (OpenClaw config gets disabled, not deleted).

### `claude-soul`
Gives identity, personality, and purpose to any Claude Code agent in any directory. Guided discovery that produces a `CLAUDE.md` with real character, seeds the native memory system, and creates a bootstrap ritual. Output is not a generic assistant — it's an agent with a name, opinions, tone, and boundaries.

---

## Install

```
/plugin marketplace add vicbullet/agent-factory
/plugin install agent-factory
```

Or manually:

```bash
git clone https://github.com/vicbullet/agent-factory.git ~/.claude/plugins/agent-factory
```

---

## Usage

### Create a new OpenClaw agent

```
/agent-factory:create-agent
```

Or with an agent name:

```
/agent-factory:create-agent archie
```

Guides you interactively through all phases.

### Migrate an OpenClaw agent to Claude Code

```
/agent-factory:migrate-openclaw-to-claude
```

Or describe the intent in natural language ("migrate `<agent>` from OpenClaw to Claude Code", "I want to pull `<agent>` off the server", "perdi os crons do OpenClaw") and the skill auto-triggers. Walks through 8 phases:

1. **Discovery / Inventory** — find canonical files, current OpenClaw state, tokens
2. **Architecture decisions** — directory layout, vault isolation, channel reuse, cron strategy
3. **Build agent home** — `CLAUDE.md` orchestrator, `BOOTSTRAP.md`, `memory/` daily logs
4. **Launch alias** — `~/.zshrc` shell alias with channel flags
5. **Telegram migration** — same bot, same conversation, manual `access.json` workaround for the `/telegram:access` bug
6. **Slack migration** — same bot identity for outbound DMs/posts (one-way; inbound stays on Telegram)
7. **Cron jobs** — `launchd` plists with shared task runner; preserves all heartbeat schedules
8. **Validation + OpenClaw shutdown** — reversible disable on the server (config backup + workspace preserved)

### Give a Claude Code agent personality

```
/agent-factory:claude-soul
```

Or trigger naturally: "I want my Claude to have personality", "set up a new agent", "dar vida a um agente". Six-phase guided discovery:

0. Understand the request and target directory
1. Who is the user? (deep profile that shapes how the agent serves)
2. Who is the agent? (identity + soul + Molty refinement for non-corporate voice)
3. Confirm operational rules (sensible defaults, tweakable)
4. Build files: `CLAUDE.md`, `BOOTSTRAP.md`, native memory seed
5. Telegram setup (mandatory — reachable from phone)
6. First contact — agent reads BOOTSTRAP.md and introduces itself in its own voice

Output is a CLAUDE.md you'd actually want to talk to at 2am, not a generic chatbot.

---

## Prerequisites

### `create-agent`
- Linux server with [OpenClaw](https://openclaw.ai) installed
- Claude Code CLI
- Model API key (OpenAI, Anthropic, etc.)
- Telegram bot token and/or Slack app tokens (for channels)

### `migrate-openclaw-to-claude`
- macOS (uses `launchd`; on Linux substitute `systemd timers`)
- Homebrew (`brew install oven-sh/bun/bun` is required — Telegram plugin needs Bun)
- SSH/cloudflared access to the OpenClaw server (to inventory state and shut down cleanly)
- Tokens for Telegram bot and Slack bot (recovered during the discovery phase)
- Claude Code CLI logged in

### `claude-soul`
- Claude Code CLI logged in
- Telegram plugin installed (`/plugin install telegram@claude-plugins-official`) — Phase 5 sets up reachability
- Bun (`brew install oven-sh/bun/bun`) — required by the Telegram plugin

---

## What's inside

### `create-agent`
Encodes 15 critical rules from real OpenClaw setup failures: `auth-profiles.json` format gotchas, Telegram allowlist trap, heartbeat per-agent override, Dream cron timezone requirement, browser automation stack (Xvfb + headed Chromium), MCP env var conventions, API silent field acceptance.

### `migrate-openclaw-to-claude`
Encodes lessons from a real OpenClaw → Claude Code migration: vault contamination via parent `.claude/CLAUDE.md`, Bun runtime requirement, `/telegram:access` ignoring `TELEGRAM_STATE_DIR`, `/schedule` remote MCP gaps, launchd PATH minimal, Mac sleep cron skips, headless browser limitations.

7 reference docs ship with the skill:

- `inventory-openclaw.md` — discovery checklist
- `vault-isolation.md` — multi-agent vault refactor pattern
- `telegram-migration.md` — preserve the bot (Bun + access.json gotchas)
- `slack-migration.md` — preserve the bot identity, with templated outbound CLI
- `launchd-cron-migration.md` — task runner + plist templates
- `openclaw-shutdown.md` — reversible disable
- `agent-home-template.md` — CLAUDE.md/BOOTSTRAP.md/memory boilerplate

### `claude-soul`
Built around the **Molty refinement** principle: every soul draft passes 8 rules before the user sees it (have opinions, delete corporate, ban sycophancy, enforce brevity, allow humor, allow pushback, allow swearing when it lands, the "2am test"). Reference docs include shallow-vs-deep user profile examples, default operational rules, the Molty prompt itself, and the Telegram setup guide.

---

## License

MIT
