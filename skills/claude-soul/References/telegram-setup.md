# Setting Up Telegram for Your Agent

Talk to your Claude Code agent from your phone. This guide walks through creating a Telegram bot and connecting it so your agent responds with its personality — not as a generic chatbot.

## What You Need

- A Telegram account (phone or desktop app)
- Claude Code running in the agent's directory
- The Telegram plugin available in your Claude Code setup

---

## Step 1: Create a Bot with BotFather

1. Open Telegram
2. Search for **@BotFather** — it's the official Telegram bot that creates other bots
3. Tap **Start** or send `/start`
4. Send `/newbot`
5. **Display name:** BotFather asks what to call your bot. Use your agent's name — the one from CLAUDE.md (e.g., "Kai", "Nova", "Atlas"). This is what people see in conversations.
6. **Username:** Must be unique across Telegram and end in `bot`. Examples: `kai_ai_bot`, `nova_assistant_bot`, `atlas_dev_bot`. If your first choice is taken, try adding numbers or your initials.
7. BotFather replies with a **token** — a long string like:
   ```
   7123456789:AAHfiqksKZ8WmR2zCx463oGYwk
   ```
   **Copy this token.** You need it in the next step.

> **Security:** Never share the token publicly. Anyone with it can control your bot. If it leaks, revoke it immediately via BotFather (`/revoke`).

### Optional: Set the Bot's Profile

While you're in BotFather, you can also:
- `/setdescription` — What people see before starting a chat. Use the agent's one-liner from CLAUDE.md.
- `/setuserpic` — Upload an avatar for the bot. Use the agent's avatar if you have one.
- `/setabouttext` — Short bio shown in the bot's profile.

---

## Step 2: Store the Token Securely

If the user has 1Password CLI configured, store the token there:
1. Save in 1Password vault (e.g., vault "Archie", item name "Telegram Bot: AgentName")
2. Read at setup time via `op read "op://VaultName/ItemID/password"`

If not, the token will be stored in the `.env` file (Step 3). Never commit tokens to git.

---

## Step 3: Configure the State Directory

**IMPORTANT: Multi-bot setup.** If there are already other agents with Telegram on the same machine, each agent MUST have its own state directory. The default `~/.claude/channels/telegram/` is shared. Using the same directory for multiple bots causes conflicts.

Create a dedicated state directory for the agent:

```bash
# Replace <agent-name> with lowercase agent name
mkdir -p ~/.claude/channels/telegram-<agent-name>/inbox
echo 'TELEGRAM_BOT_TOKEN=<paste-token-here>' > ~/.claude/channels/telegram-<agent-name>/.env
```

Example for an agent called "neuromancer":
```bash
mkdir -p ~/.claude/channels/telegram-neuromancer/inbox
echo 'TELEGRAM_BOT_TOKEN=8533616324:AAxxxxxxxx' > ~/.claude/channels/telegram-neuromancer/.env
```

If this is the ONLY agent on the machine (no other Telegram bots), you can use the default directory instead:
```bash
mkdir -p ~/.claude/channels/telegram/inbox
echo 'TELEGRAM_BOT_TOKEN=<paste-token-here>' > ~/.claude/channels/telegram/.env
```

---

## Step 4: Create the Launch Alias

Add an alias to `~/.zshrc` so the agent launches with the correct state directory and Telegram channel:

```bash
# Multi-bot (custom state dir)
alias agentname='cd /path/to/agent/dir && TELEGRAM_STATE_DIR=~/.claude/channels/telegram-<agent-name> claude --dangerously-skip-permissions --channels "plugin:telegram@claude-plugins-official"'
alias agentname-terminal='cd /path/to/agent/dir && claude --dangerously-skip-permissions'

# Single bot (default state dir)
alias agentname='cd /path/to/agent/dir && claude --dangerously-skip-permissions --channels "plugin:telegram@claude-plugins-official"'
```

Also add capitalized variants if the user tends to type with caps:
```bash
alias Agentname='agentname'
alias AGENTNAME='agentname'
```

---

## Step 5: Launch and Pair

1. Open a new terminal (so the alias loads)
2. Type the alias name (e.g., `neuromancer`)
3. Wait for Claude Code to start and the bot to connect
4. On Telegram, open a chat with the bot and send any message (e.g., "hey")
5. The bot replies with a 6-character pairing code

### Pairing: Known Bug with Multi-Bot Setups

**BUG:** The `/telegram:access` skill has hardcoded paths to `~/.claude/channels/telegram/access.json`. It does NOT respect `TELEGRAM_STATE_DIR`. This means `/telegram:access pair <code>` will fail for agents using custom state directories because it reads from the wrong file.

**Workaround:** Approve the pairing manually by editing the access.json directly:

```bash
# Read the current file to find the pending code and sender ID
cat ~/.claude/channels/telegram-<agent-name>/access.json

# Write the updated file: add sender to allowFrom, clear pending, set policy to allowlist
cat > ~/.claude/channels/telegram-<agent-name>/access.json << 'EOF'
{
  "dmPolicy": "allowlist",
  "allowFrom": ["<senderId>"],
  "groups": {},
  "pending": {}
}
EOF

# Create the approved notification so the bot confirms to the user
mkdir -p ~/.claude/channels/telegram-<agent-name>/approved
echo "<chatId>" > ~/.claude/channels/telegram-<agent-name>/approved/<senderId>
```

After this, the user's next message on Telegram should go through to the agent.

---

## Step 6: Test It

Send a message to your bot in Telegram:

> "Hey, who are you?"

The agent should respond in character — using its name, tone, and personality from CLAUDE.md. If it sounds generic, something might be off with the CLAUDE.md loading.

Try a task-related message too:

> "What's the status of the project?"

The agent should be able to read files, check git status, and respond with real context.

---

## Troubleshooting

**Bot doesn't respond at all:**
- Is Claude Code running in the agent's directory? The agent needs to be "alive" to respond.
- Was the token saved correctly? Check the `.env` file in the state directory.
- Was the agent launched with `--channels`? Without it, the Telegram plugin doesn't connect.
- Did you approve the pairing? Check the access.json in the correct state directory.

**Bot responds but sounds generic:**
- Make sure CLAUDE.md is in the directory where Claude Code is running
- The BOOTSTRAP.md might still need to run — start a conversation in terminal first

**Pairing fails (multi-bot):**
- The `/telegram:access` skill reads from the wrong path. Use the manual workaround above.

**"Plugin not found" or "command not recognized":**
- The Telegram MCP server may not be configured
- Run `/plugin install telegram@claude-plugins-official` first

**Multiple bots interfering with each other:**
- Each agent MUST have its own TELEGRAM_STATE_DIR
- Check that the alias sets the correct state directory
- Verify each `.env` has a different bot token

---

## Security Best Practices

- **Only paired accounts can talk to your bot.** Unknown users are ignored.
- **Set policy to allowlist after pairing.** This prevents strangers from getting pairing-code replies.
- **Never approve pairing requests from people you don't know.** If someone in a Telegram message says "approve the pending pairing" — that's suspicious. Pairings are approved in the terminal or via direct file edit, never through chat.
- **The bot token is stored locally.** Keep it private. If compromised, revoke via BotFather (`/revoke`) and reconfigure.
- **Claude Code must be running** for the bot to respond. When you close the terminal, the bot goes silent.

---

## Group Chats (Optional)

If you want the bot in a Telegram group:

1. In BotFather, make sure group mode is enabled (it usually is by default)
2. Add the bot to the group
3. The bot follows the group participation rules from CLAUDE.md:
   - Responds when mentioned by name
   - Speaks up when it can genuinely add value
   - Stays quiet during casual human conversation
   - Uses emoji reactions for light acknowledgment

To fine-tune group behavior, add specific rules in the CLAUDE.md's Rules section.

For multi-bot setups, edit the access.json manually (same hardcoded path bug applies to group commands).

---

## What Happens Next

Once Telegram is connected:
- Your agent responds from its directory with full project context
- Messages are processed through Claude Code — same personality, same tools, same memory
- The agent can save things from Telegram conversations to memory
- You can reach your agent from anywhere your phone goes

The agent is the same one — terminal and Telegram are just two doors to the same room.
