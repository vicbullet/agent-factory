# Migração Slack outbound (preservar identidade)

Mantém o **mesmo bot Slack** do OpenClaw mandando DMs e posts no Claude Code local. DMs aparecem como o **bot** no workspace, não como o usuário humano.

⚠️ **Limitação crítica:** Claude Code não tem plugin "channels" pra Slack (só Telegram). Por isso a migração é **outbound only**. Pra usuário falar com o agente, usa Telegram ou terminal local.

## Pré-requisitos

- [ ] Bot token (xoxb-...) recuperado da Fase 0
- [ ] Agente OpenClaw desligado OU verificar que ambos podem coexistir (Slack permite múltiplos posts do mesmo bot ID)

## Passo 1: Validar token + descobrir identity

```bash
TOKEN="<xoxb-... do inventário>"
curl -s -H "Authorization: Bearer $TOKEN" "https://slack.com/api/auth.test" | python3 -m json.tool
```

Saída esperada:

```json
{
  "ok": true,
  "url": "https://<workspace>.slack.com/",
  "team": "<team name>",
  "user": "<bot username>",
  "team_id": "T...",
  "user_id": "U...",
  "bot_id": "B...",
  ...
}
```

Anote `user_id` (esse é quem o bot vai aparecer como), `team_id`, `team`.

## Passo 2: Salvar credenciais

```bash
AGENT="<agentname>"  # lowercase
mkdir -p ~/.claude/channels/slack-$AGENT
cat > ~/.claude/channels/slack-$AGENT/.env <<EOF
SLACK_BOT_TOKEN=$TOKEN
SLACK_BOT_USER_ID=<U... do auth.test>
SLACK_BOT_NAME=<bot name>
SLACK_TEAM_ID=<T... do auth.test>
SLACK_TEAM_DOMAIN=<workspace domain>
EOF
chmod 600 ~/.claude/channels/slack-$AGENT/.env
```

## Passo 3: Helper CLI `<agentname>-slack-dm`

Crie `~/.local/bin/<agentname>-slack-dm` (substitui `<agentname>`):

```bash
#!/usr/bin/env bash
# <agentname>-slack-dm — manda DM/post no Slack como o bot
#
# Uso:
#   <agentname>-slack-dm <user_id|@username> "<msg>"
#   <agentname>-slack-dm --channel <channel_id|#name> "<msg>"

set -euo pipefail

ENV_FILE="$HOME/.claude/channels/slack-<agentname>/.env"
[[ ! -f "$ENV_FILE" ]] && { echo "ERR: $ENV_FILE não existe" >&2; exit 2; }
source "$ENV_FILE"
[[ -z "${SLACK_BOT_TOKEN:-}" ]] && { echo "ERR: SLACK_BOT_TOKEN ausente" >&2; exit 2; }

usage() {
  cat <<EOF >&2
Uso: $0 <user_id|@username> "<msg>" | $0 --channel <channel_id|#name> "<msg>"
EOF
  exit 1
}

[[ $# -lt 2 ]] && usage
mode="user"
if [[ "$1" == "--channel" ]]; then mode="channel"; shift; fi
target="$1"; shift; message="$*"

# Resolve @username → user_id
if [[ "$mode" == "user" && "$target" == @* ]]; then
  username="${target#@}"
  target=$(curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    "https://slack.com/api/users.list" \
    | python3 -c "
import json, sys, os
data = json.load(sys.stdin)
if not data.get('ok'): sys.exit(0)
for u in data.get('members', []):
    if u.get('name') == os.environ['U'] or u.get('profile', {}).get('display_name') == os.environ['U']:
        print(u['id']); break
" U="$username")
  [[ -z "$target" ]] && { echo "ERR: @$username não encontrado" >&2; exit 3; }
fi

# Resolve #channel → channel_id
if [[ "$mode" == "channel" && "$target" == \#* ]]; then
  channelname="${target#\#}"
  target=$(curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    "https://slack.com/api/conversations.list?limit=1000&types=public_channel,private_channel" \
    | python3 -c "
import json, sys, os
data = json.load(sys.stdin)
if not data.get('ok'): sys.exit(0)
for c in data.get('channels', []):
    if c.get('name') == os.environ['C']:
        print(c['id']); break
" C="$channelname")
  [[ -z "$target" ]] && { echo "ERR: #$channelname não encontrado" >&2; exit 3; }
fi

# DM: abre conversation
if [[ "$mode" == "user" ]]; then
  conv=$(curl -s -X POST \
    -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"users\":\"$target\"}" \
    "https://slack.com/api/conversations.open" \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['channel']['id'] if d.get('ok') else '')")
  [[ -z "$conv" ]] && { echo "ERR ao abrir DM" >&2; exit 3; }
  target="$conv"
fi

# chat.postMessage (JSON via Python pra evitar quoting)
payload=$(python3 -c 'import json, sys; print(json.dumps({"channel": sys.argv[1], "text": sys.argv[2]}))' "$target" "$message")
result=$(curl -s -X POST \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$payload" \
  "https://slack.com/api/chat.postMessage")

ok=$(echo "$result" | python3 -c "import json,sys; d=json.load(sys.stdin); print('yes' if d.get('ok') else d.get('error','unknown'))")

if [[ "$ok" == "yes" ]]; then
  ts=$(echo "$result" | python3 -c "import json,sys; print(json.load(sys.stdin).get('ts',''))")
  echo "✅ enviado para $target (ts=$ts)"
  exit 0
else
  echo "ERR Slack: $ok" >&2
  exit 3
fi
```

⚠️ Substitua **TODAS** as ocorrências de `<agentname>` por o nome real do agente em lowercase.

```bash
chmod +x ~/.local/bin/<agentname>-slack-dm
```

## Passo 4: Test

```bash
# DM real pro usuário (substitui USER_ID pelo Slack user_id real)
<agentname>-slack-dm <USER_ID> "🐝 test from <agent> — outbound funcionando"
```

Output esperado:
```
✅ enviado para D0... (ts=...)
```

Confirma com o usuário que recebeu DM **como o bot** (não como auto-DM dele mesmo).

## Passo 5: Documentar no CLAUDE.md do agente

Adiciona uma seção "Slack outbound" no CLAUDE.md do agente:

```markdown
## Slack outbound

Você tem identidade própria como bot `<bot_name>` (`<bot_user_id>`) no workspace `<team>`. Pra mandar mensagem via Bash:

# DM para usuário (resolve @username automaticamente)
<agentname>-slack-dm <user_id> "mensagem"
<agentname>-slack-dm @<username> "mensagem"

# Post em canal
<agentname>-slack-dm --channel <channel_id> "mensagem"
<agentname>-slack-dm --channel #<channel_name> "mensagem"

User IDs principais:
- <Owner>: <user_id>
- <Outros relevantes>: <user_ids>

Canais relevantes (do HEARTBEAT.md):
- <C...>: <descrição do canal>
- <C...>: <descrição>

Token e config em ~/.claude/channels/slack-<agentname>/.env. Não inclua o token em outputs.
```

Isso é crítico — sem documentar, o agente headless (em cron jobs) não sabe que tem essa CLI disponível.

## Multi-agente Slack na mesma máquina

Cada agente tem:
- Bot token próprio
- `~/.claude/channels/slack-<agentname>/` próprio
- `<agentname>-slack-dm` próprio

Não compartilha. Cada bot tem identidade própria no workspace.

## Inbound (Slack → Beyoncé)

Não tem solução pronta no Claude Code. Workarounds:
1. **Telegram** como canal de inbound (já temos plugin pronto)
2. **Webhook server custom** que escuta eventos Slack e injeta no agente (complexo, não cobre essa skill)
3. **Slack RSS / Slack API polling** rodando local que cria daily logs lidos pelo agente na próxima sessão (semi-realtime)

Pra esta skill, **outbound-only** é suficiente: agente notifica/posta proativamente, usuário fala com ela via Telegram ou terminal.

## Segurança

- Token chmod 600
- Nunca em git, nunca em CLAUDE.md (mesmo o snippet documentado acima é só sobre o caminho do .env, não o valor)
- Se vazar: revoga em api.slack.com/apps → escolhe app → "Install App" → "Reinstall to Workspace" (gera novo token)
- Bot só posta em canais onde foi adicionado — adicione previamente nos canais do HEARTBEAT
