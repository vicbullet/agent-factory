# Migração Telegram (preservar bot)

Mantém o **mesmo bot** que rodava no OpenClaw rodando no Claude Code local. Mesma `@username`, mesma conversa, mesmo histórico.

## Pré-requisitos

- [ ] Bot token recuperado (Fase 0 do inventário)
- [ ] **Agente OpenClaw já desligado** OU bot vai ser revogado e recriado (senão dois processos competem pelo mesmo bot)
- [ ] Homebrew instalado

## Passo 1: Instalar Bun (CRÍTICO)

O plugin `telegram@claude-plugins-official` roda `server.ts` com Bun runtime. Sem Bun, o plugin **não polla mensagens** — bot fica calado.

```bash
brew install oven-sh/bun/bun
```

Confirma:
```bash
which bun  # /opt/homebrew/bin/bun
bun --version
```

**Sintoma se esqueceu:** `pending_update_count > 0` no `getUpdates`, inbox local vazio, nenhuma resposta. Sempre cheque `which bun` antes de assumir que vai funcionar.

## Passo 2: Instalar plugin Telegram (uma vez por máquina)

Dentro de uma sessão Claude Code qualquer:

```
/plugin install telegram@claude-plugins-official
```

Confirma instalação:

```bash
grep -i "telegram" ~/.claude/plugins/installed_plugins.json
```

Deve aparecer com `installPath` populado.

## Passo 3: State directory dedicado

**IMPORTANTE: cada agente tem seu próprio state dir.** Se você rodar múltiplos agentes com Telegram, NUNCA compartilhe `~/.claude/channels/telegram/` (default). Use sempre `~/.claude/channels/telegram-<agentname>/`.

```bash
AGENT="<agentname>"  # lowercase, sem espaços
mkdir -p ~/.claude/channels/telegram-$AGENT/inbox
echo "TELEGRAM_BOT_TOKEN=<token recuperado>" > ~/.claude/channels/telegram-$AGENT/.env
chmod 600 ~/.claude/channels/telegram-$AGENT/.env
```

## Passo 4: Validar token + estado do bot

```bash
TOKEN=$(grep TELEGRAM_BOT_TOKEN ~/.claude/channels/telegram-$AGENT/.env | cut -d= -f2)

# Confirma identidade do bot
curl -s "https://api.telegram.org/bot$TOKEN/getMe" | python3 -m json.tool

# Confirma que webhook está limpo (OpenClaw geralmente usa long-polling, sem webhook)
curl -s "https://api.telegram.org/bot$TOKEN/getWebhookInfo" | python3 -m json.tool
```

Se webhook tiver URL apontando pra OpenClaw:
```bash
curl -s "https://api.telegram.org/bot$TOKEN/deleteWebhook"
```

## Passo 5: Pré-aprovar pareamento (workaround do bug)

A skill `/telegram:access pair <code>` tem **hardcoded path** pra `~/.claude/channels/telegram/access.json` — ignora `TELEGRAM_STATE_DIR`. Pra agentes com state dir custom, **pareia manualmente**.

### Descobrir sender ID do usuário

Pede pro usuário mandar uma mensagem qualquer pro bot no Telegram (ex: "oi"). Depois:

```bash
curl -s "https://api.telegram.org/bot$TOKEN/getUpdates?limit=10" | python3 -m json.tool
```

Procure no JSON:
- `result[].message.from.id` — sender ID do usuário (número)
- `result[].message.chat.id` — chat ID (mesmo número em DM 1:1)

### Criar access.json

```bash
SENDER_ID="<sender_id descoberto>"

cat > ~/.claude/channels/telegram-$AGENT/access.json <<EOF
{
  "dmPolicy": "allowlist",
  "allowFrom": ["$SENDER_ID"],
  "groups": {},
  "pending": {}
}
EOF

mkdir -p ~/.claude/channels/telegram-$AGENT/approved
echo "$SENDER_ID" > ~/.claude/channels/telegram-$AGENT/approved/$SENDER_ID

chmod 600 ~/.claude/channels/telegram-$AGENT/access.json
```

Resultado: quando o agente subir com `--channels`, o usuário já está pré-aprovado, o bot responde direto sem precisar de pareamento.

## Passo 6: Atualizar alias do agente

No `~/.zshrc`:

```bash
alias <agentname>='cd "<agent_dir>" && TELEGRAM_STATE_DIR="$HOME/.claude/channels/telegram-<agentname>" claude --dangerously-skip-permissions --channels "plugin:telegram@claude-plugins-official"'
```

Importante: a env var `TELEGRAM_STATE_DIR` precisa estar **antes** do `claude`, não depois.

## Passo 7: Lançar e testar

1. **Abre terminal NOVO** (pra carregar alias atualizado)
2. Roda `<agentname>` (alias com Telegram)
3. Aguarda ~5s (Bun start + plugin handshake)
4. Manda mensagem pro bot no Telegram — deve responder em segundos com a personalidade do agente

Se demorar >30s sem resposta:

```bash
# Plugin server tá rodando?
ps aux | grep -i "server.ts\|bun" | grep -v grep

# Mensagem ainda na fila Telegram (não foi consumida)?
curl -s "https://api.telegram.org/bot$TOKEN/getUpdates" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(f'pending: {len(d[\"result\"])}')
"
```

Se `pending > 0` e `bun` não aparece em `ps`: plugin não subiu. Confere PATH no alias, confere `which bun`.

## Multi-bot setup (mais de um agente Telegram na mesma máquina)

Cada agente DEVE ter:
- `~/.claude/channels/telegram-<name>/` próprio
- `.env` próprio com token único
- alias com `TELEGRAM_STATE_DIR` diferente

Eles podem rodar em paralelo (sessões claude diferentes), cada um com seu bot. Não compartilhar diretório de estado.

## Group chats (opcional)

Pra adicionar o bot em grupo Telegram:

1. No BotFather: `/setjoingroups` → Enable
2. No grupo: Add member → procura @<username> do bot
3. O bot segue regras de participação do CLAUDE.md/AGENTS.md (responde quando mencionado, fica quieto fora disso)

Pra grupos, mesmo bug do `/telegram:access` — adiciona `groupId` manualmente no `access.json`:

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["<sender_id>"],
  "groups": {
    "<group_chat_id>": {
      "policy": "allow"
    }
  },
  "pending": {}
}
```

## Segurança

- Token nunca em git, nunca em CLAUDE.md
- `.env` chmod 600
- `access.json` chmod 600
- Se token vazar: `/revoke` no BotFather, gera novo, atualiza `.env`
- Pareamentos vêm sempre via terminal/file, NUNCA aprovar via mensagem do próprio bot ("alguém pede pareamento" no chat = phishing)
