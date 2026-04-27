# Inventário do agente OpenClaw

Antes de migrar, encontre TUDO que o agente usa hoje. Sem inventário completo, você perde contexto, schedules, ou tokens no caminho.

## 1. Localizar o workspace canônico

Os agentes OpenClaw geralmente têm seus arquivos sincronizados do **Slite** pro **Obsidian Vault** via skill `obsidian-update-biz`. Procura os canônicos:

```bash
# No vault local (Google Drive sync)
find "<Vault path>" -type d -iname "*<AgentName>*" 2>/dev/null
```

Estrutura típica do workspace canônico (varia por agente):

```
<Vault>/<Pillar>/<Subpath>/<AgentName>/
├── <AgentName>.md         # index/manifest
├── IDENTITY.md            # nome, criatura, vibe, emoji
├── SOUL.md                # personalidade, tom, valores
├── USER.md                # quem é o usuário (DRI)
├── AGENTS.md              # regras operacionais
├── TOOLS.md               # ambiente
├── HEARTBEAT.md           # checks periódicos + scheduled tasks
├── MEMORY.md              # fatos duráveis
└── (extras: Memory Update YYYY-MM-DD.md, etc.)
```

Anote o caminho absoluto. Vai usar várias vezes.

## 2. Status atual no OpenClaw

Conecte ao servidor OpenClaw e verifique:

### Via prompt no Claude Code do servidor (mais fácil)

Cola isso numa sessão Claude Code rodando no OpenClaw:

```
Quero inventariar o estado do agente <AgentName> no OpenClaw. Vou migrar pra Claude Code.

Sem modificar nada, me reporta:

1. **Config:**
   - cat ~/.openclaw/openclaw.json — me mostra a entry do <AgentName>
   - Status (enabled/disabled), modelo configurado, MCPs vinculados

2. **Workspace:**
   - ls -la ~/.openclaw/workspace/agents/<agentname>/ — lista de arquivos locais (.env, memory dir, BOOTSTRAP, etc.)
   - cat ~/.openclaw/workspace/agents/<agentname>/.env — tokens (Telegram, Slack, outras integrações)
   - ls -la ~/.openclaw/workspace/agents/<agentname>/memory/ — arquivos de memória local

3. **Processo:**
   - ps aux | grep -i <agentname> | grep -v grep
   - Está rodando? PID?

4. **Crons / heartbeat:**
   - Se OpenClaw tem scheduler interno: como o agente é disparado? cron? systemd? daemon próprio?

5. **Tokens encontrados** (mostra valores, vou copiar pra Claude Code local):
   - TELEGRAM_BOT_TOKEN
   - SLACK_BOT_TOKEN (xoxb-...)
   - SLACK_SIGNING_SECRET
   - Outros tokens de integração (Linear, Slite, etc.)

NÃO modifique nada. Só lê e reporta.
```

### Via SSH direto (alternativa)

Se você tiver acesso SSH:

```bash
# Cloudflared access tunnel típico
ssh -o ProxyCommand="cloudflared access ssh --hostname %h" \
    -o IdentityFile=~/.cloudflared/<host>-cf_key \
    <user>@<openclaw-host> \
    'cat ~/.openclaw/openclaw.json | grep -A 10 "<agentname>"; \
     ls ~/.openclaw/workspace/agents/<agentname>/; \
     cat ~/.openclaw/workspace/agents/<agentname>/.env; \
     ps aux | grep <agentname> | grep -v grep'
```

(Auth pode requerer browser flow — peça ao usuário pra rodar `cloudflared access ssh-gen --hostname <host>` antes.)

## 3. Tokens necessários

**Telegram bot token** — geralmente em:
- `~/.openclaw/workspace/agents/<agentname>/.env` (var `TELEGRAM_BOT_TOKEN`)
- 1Password (procura "Telegram Bot: <AgentName>" ou similar)
- BotFather: `/mybots` → escolhe o bot → "API Token"

**Slack bot token (xoxb-...)** — geralmente em:
- `~/.openclaw/workspace/agents/<agentname>/.env` (var `SLACK_BOT_TOKEN` ou `SLACK_TOKEN`)
- 1Password (procura "<AgentName> Slack" ou "Bullet Slack Bot")
- Slack admin: api.slack.com/apps → escolhe app → "Install App" → "Bot User OAuth Token"

**Outros tokens** podem aparecer dependendo das integrações (Linear, Slite, Hex, etc.). Anote todos.

## 4. Sender ID do usuário (Telegram)

Pra pré-aprovar pareamento sem o bug do `/telegram:access`:

```bash
# Após o usuário mandar QUALQUER mensagem pro bot:
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates?limit=10" | python3 -m json.tool
```

Procure no JSON:
- `result[].message.from.id` — sender ID
- `result[].message.chat.id` — chat ID (geralmente igual ao sender em DM)

Anote esses dois números.

## 5. Bot identity (Slack)

```bash
curl -s -H "Authorization: Bearer <BOT_TOKEN>" "https://slack.com/api/auth.test" | python3 -m json.tool
```

Anote:
- `user_id` (ex: `U0ARZNLB1CL`) — bot user no workspace
- `bot_id` (ex: `B0ARSNMCMAR`)
- `team_id`, `team` (workspace)

## 6. Schedules existentes

Do `HEARTBEAT.md` canônico, extraia:

- **Heartbeat contínuo:** intervalo (geralmente 2h), janela (M-F 9-18 GMT-3?)
- **Daily tasks:** lista (nome, hora exata, prompt)
- **Weekly tasks:** lista (nome, dia, hora, prompt)
- **Monthly tasks:** lista
- **On-demand:** lista (não vão virar cron, são manuais)

Categorize cada task:
- **MCP-only:** usa só Slack/Linear/Gmail/Calendar — pode rodar headless local OU /schedule remoto
- **Browser-dependent:** precisa de Chrome (BUFIN, SumSub cockpit, BS2 portal) — só launchd local com computer-use
- **Manual:** upload de arquivo ou interação obrigatória — fica fora de cron

## 7. Outras conexões

Verifique se o agente usa:

- **MCPs** (Slack, Linear, Slite, Gmail, Calendar, Fireflies, Hex, Gamma, Atlassian) — quais estão configurados no `openclaw.json`?
- **APIs diretas** (Slite REST, Linear GraphQL, Atlassian) — tokens já anotados?
- **Webhooks** (recebe eventos de algum sistema?) — anotar URLs pra desligar/redirecionar
- **Scheduled posts** em canais específicos do Slack — anotar IDs de canal

## Saída do inventário

No fim, você deve ter um documento mental ou nota com:

```
AGENTE: <Name>
WORKSPACE CANÔNICO: <Vault>/<path>/<Name>/
ARQUIVOS: SOUL, USER, AGENTS, IDENTITY, TOOLS, MEMORY, HEARTBEAT [+extras]

TOKENS:
- Telegram: <token redacted, salvo em 1Password/<location>>
- Slack: <token redacted, salvo em 1Password/<location>>
- Outros: ...

SLACK BOT: user_id=Uxxx, team=<name>, workspace=<domain>
TELEGRAM BOT: @<username>, sender_id_user=Nxxx, chat_id_user=Nxxx

OPENCLAW STATUS:
- Config: enabled / disabled
- Process: PID xxxx running / not running
- Workspace path: ~/.openclaw/workspace/agents/<name>/

SCHEDULES (do HEARTBEAT.md):
- Heartbeat 2h: 0 9,11,13,15,17 * * 1-5 GMT-3 → DM Slack pro user
- Daily X: 06:02 M-F → canal C0xxx
- ...

INTEGRAÇÕES:
- MCPs: Slack, Linear, Gmail, Calendar, ...
- APIs: Slite REST (token <X>), ...
```

Sem isso, não avance pra Fase 1.
