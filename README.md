# OpenClaw Agent Factory

Claude Code plugin for the full OpenClaw agent lifecycle — create new agents from scratch, or migrate existing ones to Claude Code.

## Skills included

### `create-agent`
Creates fully operational [OpenClaw](https://openclaw.ai) AI agents from scratch via guided 10-phase setup.

### `migrate-openclaw-to-claude`
Migrates an existing OpenClaw agent (remote 24/7 daemon) to Claude Code (local terminal + headless cron via launchd), preserving identity, Telegram bot, Slack bot, and schedules. Same bots, same conversations, same schedule times — different infra.

---

## Install

```bash
claude plugin marketplace add vicbullet/openclaw-agent-factory
claude plugin install openclaw-agent-factory
```

Or manually:

```bash
git clone https://github.com/vicbullet/openclaw-agent-factory.git ~/.claude/plugins/openclaw-agent-factory
```

---

## Usage

### Create a new OpenClaw agent

```
/openclaw-agent-factory:create-agent
```

Or with an agent name:

```
/openclaw-agent-factory:create-agent archie
```

Guides you interactively through all phases:

- **Identity** — SOUL.md, USER.md, IDENTITY.md, AGENTS.md (personality, capabilities, rules)
- **Model Auth** — API key configuration for OpenAI, Anthropic, Google, etc.
- **Channels** — Telegram (allowlist) and Slack (pairing) with routing bindings
- **Cortex v2 Memory** — Persistent structured memory with topic files, scoring, and hybrid search
- **Dream Pipeline** — Nightly 12-step consolidation of conversations into long-term memory
- **Self-Improving** — Automatic capture of errors, corrections, and feature requests
- **MCP Integrations** — npm packages + custom MCP servers for any API
- **Browser Automation** — Xvfb + Chromium for web UI operations the API doesn't support

### Migrate an OpenClaw agent to Claude Code

```
/openclaw-agent-factory:migrate-openclaw-to-claude
```

Or just describe the intent: "migrate `<agent>` from OpenClaw to Claude Code", "I want to pull `<agent>` off the server", etc.

Walks through 8 phases:

1. **Discovery / Inventory** — find canonical files, current OpenClaw state, tokens
2. **Architecture decisions** — where the agent lives, vault isolation, channel reuse, cron strategy
3. **Build agent home** — `CLAUDE.md` orchestrator, `BOOTSTRAP.md`, `memory/` daily logs
4. **Launch alias** — `~/.zshrc` shell alias with channel flags
5. **Telegram migration** — same bot, same conversation, manual access.json workaround for the `/telegram:access` bug
6. **Slack migration** — same bot identity for outbound DMs/posts (one-way; inbound stays on Telegram)
7. **Cron jobs** — `launchd` plists with shared task runner; preserves all heartbeat schedules
8. **Validation + OpenClaw shutdown** — reversible disable on the server (config backup + workspace preserved)

Preserves:

- ✅ Identity (`SOUL.md`, `USER.md`, `AGENTS.md`, `MEMORY.md`, `HEARTBEAT.md` canonical files)
- ✅ Telegram channel (same bot token → same `@username` → same conversation history)
- ✅ Slack outbound (same bot identity in the workspace)
- ✅ Schedules (heartbeat + scheduled tasks at the same local times)
- ✅ Daily logs (`memory/YYYY-MM-DD.md` append-only kept)
- 🛡️ Vault multi-agent isolation (no identity leak across subfolders)
- 🔄 Reversible OpenClaw shutdown (config disabled, not deleted)

---

## Prerequisites

### For `create-agent`

- Linux server with [OpenClaw](https://openclaw.ai) installed
- Claude Code CLI
- Model API key (OpenAI, Anthropic, etc.)
- Telegram bot token and/or Slack app tokens (for channels)

### For `migrate-openclaw-to-claude`

- macOS (the skill uses `launchd`; on Linux you'd swap for `systemd timers` — same idea, different syntax)
- SSH/cloudflared access to the OpenClaw server (to inventory state and shut down cleanly)
- Homebrew (`brew install oven-sh/bun/bun` is required — the Telegram plugin needs Bun runtime)
- Claude Code CLI logged in
- Tokens for Telegram bot and Slack bot (recovered during the migration's discovery phase)

---

## What's inside

### `create-agent`

Encodes lessons learned from building production OpenClaw agents:

- 15 critical rules from real setup failures
- Correct `auth-profiles.json` format (common gotcha: `key` not `apiKey`, `type` not `mode`)
- Telegram allowlist trap (empty `allowFrom` = silent message drop)
- Heartbeat per-agent override (adding to one agent silently disables others)
- Dream cron timezone requirement (without explicit tz, runs in UTC)
- Browser automation stack (Xvfb + headed Chromium + cdpUrl — headless fails for SPAs)
- MCP env var names (always check README — APIs vary)
- API silent field acceptance (some APIs return "success" but don't persist unknown fields)

### `migrate-openclaw-to-claude`

Encodes lessons learned from a real OpenClaw → Claude Code migration:

- Vault contamination: a parent `.claude/CLAUDE.md` with agent identity leaks to every subdir agent
- Telegram plugin requires Bun runtime (silent failure mode if missing)
- `/telegram:access` skill ignores `TELEGRAM_STATE_DIR` (manual `access.json` workaround)
- `/schedule` remote agents lack Slack/Linear MCPs by default (forces local launchd for full coverage)
- launchd uses minimal PATH (must export in script)
- Bot tokens shouldn't go in remote routine prompts (cloud config exposure)
- Mac dormindo: launchd does NOT replay missed cron fires (heartbeat-style 2h schedules tolerate this; daily 06:02 jobs may silently miss)
- `claude --print` headless cannot drive a browser — tasks needing Chrome (BUFIN portal, KYC cockpit, etc.) need either `computer-use` API or stay manual

7 reference docs ship with the skill:

- `inventory-openclaw.md` — discovery checklist for the OpenClaw side (tokens, processes, schedules)
- `vault-isolation.md` — multi-agent vault refactor pattern
- `telegram-migration.md` — preserve the Telegram bot, including Bun + access.json gotchas
- `slack-migration.md` — preserve the Slack bot, with a templated `<agent>-slack-dm` outbound CLI
- `launchd-cron-migration.md` — task runner + plist templates + Mac-asleep caveat
- `openclaw-shutdown.md` — reversible disable on the server (config backup + workspace preserved)
- `agent-home-template.md` — boilerplate CLAUDE.md, BOOTSTRAP.md, memory dir

---

## License

MIT
