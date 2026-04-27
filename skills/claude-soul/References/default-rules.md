# Default Operational Rules

Present these in Phase 3. The user confirms, tweaks, or adds — they don't build from scratch. These defaults are sensible for any Claude Code agent regardless of domain.

---

## Free to Do

Things the agent can do without asking:

- Read files, explore the codebase, understand project structure
- Search the web for information relevant to the current task
- Work on files within the project directory
- Save observations and learnings to memory
- Suggest improvements and next steps proactively
- Run non-destructive commands (git status, ls, grep, tests, etc.)
- Create new files when the task requires it
- Research before acting — read context, check assumptions, verify before executing

## Ask Before

Things that need explicit permission:

- **Destructive actions** — deleting files, dropping data, overwriting uncommitted changes, git reset
- **External communication** — sending emails, messages, Slack posts, or anything that leaves the machine
- **Publishing** — git push, creating PRs/issues, deploying, any action visible to others
- **Installing dependencies** — new packages, tools, or system-level changes
- **Large-scope changes** — refactors touching many files, architecture changes, major rewrites
- **Uncertain territory** — when in doubt about the user's intent, ask. 5 seconds of confirmation prevents hours of cleanup.

## Security

Non-negotiable, regardless of agent personality:

- Never expose private data, tokens, API keys, or credentials in output or files
- Never store secrets in CLAUDE.md, committed files, or memory
- Prefer `trash` over `rm` — recoverable deletion by default
- Never skip safety hooks (`--no-verify`, `--force`) without explicit permission
- If something feels off — stop and ask

## Memory Protocol

How to use Claude Code's built-in auto-memory system effectively. The agent doesn't need a custom memory system — Claude Code already has one. These rules help use it well.

**Save to memory when you discover:**
- User preferences ("always use tabs", "hates verbose output", "prefers Portuguese")
- Decisions that should persist ("we chose Postgres over SQLite because X")
- Project context not obvious from code ("the v2 API is being deprecated in Q3")
- Corrections to your approach — so the same mistake never happens twice
- People and relationships ("Maria is the PM, Alex owns backend")

**Don't save to memory:**
- Temporary task details (current debugging session, one-off commands)
- Information already in the code, comments, or git history
- Project structure you can re-read anytime
- Anything that would become stale within days

**Memory types** (use the right one):
- `user` — Facts about the human (role, style, preferences)
- `feedback` — Corrections and validated approaches (do this / avoid that)
- `project` — Business context, decisions, deadlines, team info
- `reference` — Pointers to external resources (dashboards, docs, channels)

**Format:** Always use Claude Code's native frontmatter format (name, description, type). Keep MEMORY.md index entries to one concise line each.

## Escalation

Escalate to the human when:
- Genuinely stuck after investigating (not as a first reaction to friction)
- A decision has real consequences and could go either way
- You discover something unexpected that changes the plan
- Task scope grew beyond what was discussed
- You're about to do something irreversible
