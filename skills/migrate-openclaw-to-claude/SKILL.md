---
name: migrate-openclaw-to-claude
description: >
  Migrate an OpenClaw agent (remote 24/7 daemon) to Claude Code (local terminal + headless cron),
  preserving identity, channels (Telegram bot, Slack bot), and schedules. Use whenever you need to move
  ANY OpenClaw agent — by name or otherwise — to Claude Code infra, OR when the user says "migrate
  agent from OpenClaw", "pull agent off the server", "agent X is performing badly on OpenClaw, bring
  it to Claude Code", "I lost the OpenClaw crons", "need to recreate the heartbeat here", or any
  variation. Agent-agnostic: works for any OpenClaw agent whose canonical files (SOUL, USER, AGENTS,
  HEARTBEAT, etc.) are accessible. Focus is on keeping channels and connections intact: same Telegram
  bot, same Slack bot identity, same schedule times. Trigger on Portuguese variations too: "migrar
  agente do OpenClaw", "tirar agente do servidor", "perdi os crons", "preciso recriar o heartbeat".
---

# migrate-openclaw-to-claude

Migra um agente OpenClaw → Claude Code preservando:

- ✅ **Identidade** (SOUL, USER, AGENTS, MEMORY, HEARTBEAT canônicos)
- ✅ **Canal Telegram** (mesmo bot token, mesmo @username, conversa antiga continua)
- ✅ **Canal Slack outbound** (mesmo bot identity, mesmas DMs aparecem como o bot)
- ✅ **Schedules** (heartbeat 2h + scheduled tasks com mesmos horários)
- ✅ **Daily logs** (memory/YYYY-MM-DD.md append-only mantido)

E também:

- 🛡️ **Isola identidade no vault** (cada agente em diretório próprio, sem contaminar outros agentes que rodem em subpastas do mesmo vault)
- 🔄 **Rollback fácil** (config OpenClaw é desativada, não deletada — basta re-ativar)

## Quando usar

- Usuário disse que vai migrar um agente do OpenClaw
- Performance ruim no OpenClaw (limite ChatGPT, latência, daemon travado)
- Quer aproveitar Max plan no Claude Code em vez do plan pago do OpenClaw
- Quer rodar local com integração nativa (Telegram, Slack outbound, MCPs)

## Pré-requisitos

Antes de começar, valide:

- [ ] Acesso SSH/Cloudflared ao servidor OpenClaw (pra ler config e desligar)
- [ ] Acesso ao 1Password ou ao arquivo `.env` do agente (pra recuperar Slack bot token)
- [ ] Bot token do Telegram (geralmente em 1Password também ou no `.env` do agente)
- [ ] Workspace canônico do agente (geralmente sincronizado do Slite pro Obsidian Vault)
- [ ] Homebrew instalado (`brew --version`)
- [ ] Claude Code instalado e logado

## Fluxo geral

A migração tem **8 fases**. Faz uma de cada vez, não pula. Cada fase tem checkpoints — confirma com o usuário antes de avançar quando indicado.

### Fase 0: Discovery / Inventário

Antes de mexer em qualquer coisa, **encontre os arquivos canônicos do agente** e o **estado atual no OpenClaw**.

Leia [References/inventory-openclaw.md](References/inventory-openclaw.md) para o roteiro completo.

Saída esperada:
- Caminho do workspace canônico (ex: `Bullet Vault/05 Business/Automation/BIZ 2.0/<Pillar>/<AgentName>/`)
- Lista de arquivos canônicos (SOUL, USER, AGENTS, IDENTITY, TOOLS, MEMORY, HEARTBEAT, e variações)
- Status no OpenClaw (rodando? desativado?)
- Bot tokens (Telegram, Slack) localizados ou identificados onde buscar

### Fase 1: Decisões arquiteturais

Antes de criar arquivos, alinhe com o usuário:

1. **Onde o agente vai morar?** Recomenda `<Vault>/04 Skynet/Agents/<AgentName>/` se o vault já tem essa estrutura. Senão, fora do vault em `~/<org>-agents/<agentname>/`.

2. **O vault tem CLAUDE.md no root?** Se sim, é uma armadilha — qualquer agente em subpasta herda essa identidade. Aplique a [refatoração de isolamento](References/vault-isolation.md) ANTES de criar o novo agente. Coordene com o owner do agente que está no root (DM no Slack avisando da mudança e do novo alias dele).

3. **Telegram: reaproveitar bot ou criar novo?**
   - Reaproveitar (preferido): mesma conversa do OpenClaw continua, time já reconhece
   - Criar novo: setup limpo, mas perde histórico

4. **Slack: reaproveitar bot ou criar novo?**
   - Reaproveitar (preferido): identidade do bot preservada
   - Criar novo: scopes limpos, mas re-adicionar em canais

5. **Cron jobs: launchd local ou /schedule remoto?**
   - **launchd local** (recomendado pra agentes que dependem de skills locais, browser, ou MCP só local): roda quando Mac está ligado
   - **/schedule remoto**: roda 24/7 na nuvem Anthropic, mas só tem MCPs conectados em claude.ai/customize/connectors (geralmente Gmail/Calendar/Drive — Slack/Linear não estão por padrão)
   - **Híbrido**: tasks MCP-only via /schedule, tasks com browser/skill local via launchd

### Fase 2: Build do home do agente

Crie a estrutura no diretório escolhido:

```
<agent_dir>/
├── CLAUDE.md         # orchestrator fino apontando pros canônicos
├── BOOTSTRAP.md      # ritual de primeiro contato, auto-deleta
└── memory/           # daily logs YYYY-MM-DD.md
```

**CLAUDE.md** deve ser fino (orchestrator), apontando para os arquivos canônicos:

```markdown
# <Name> <Emoji>

> <one-liner do agente>

## Toda sessão (boot ritual)

Lê na ordem (sem pedir permissão):

1. SOUL.md — quem eu sou
2. USER.md — quem eu ajudo
3. AGENTS.md — regras operacionais
4. memory/ local — logs diários recentes
5. MEMORY.md — fatos duráveis (sessão privada)
6. HEARTBEAT.md — checks periódicos

Os arquivos canônicos vivem em:
<absolute path do workspace canônico>

São sincronizados do Slite via skill `obsidian-update-biz`. Não duplicar localmente.

## Memory local

Daily logs em `<agent_dir>/memory/YYYY-MM-DD.md` — append-only.

## Regras críticas

[copia os pontos non-negotiable de SOUL.md/AGENTS.md, mas resumido]

## Stack

[lista de ferramentas, incluindo a Slack outbound CLI quando configurada]

## Heartbeat e scheduled tasks

[resume de HEARTBEAT.md com cronograma]
```

**BOOTSTRAP.md** orienta a primeira conversa: ler arquivos canônicos, se apresentar com personalidade, oferecer config de canais que faltarem, deletar o BOOTSTRAP no fim.

**memory/** começa vazio. A própria agente preenche.

Veja exemplo completo em [References/agent-home-template.md](References/agent-home-template.md).

### Fase 3: Alias de launch

Adicione ao `~/.zshrc`:

```bash
# <Agent name> — terminal-only (sem canais)
alias <agentname>-terminal='cd "<agent_dir>" && claude --dangerously-skip-permissions'

# <Agent name> — com Telegram
alias <agentname>='cd "<agent_dir>" && TELEGRAM_STATE_DIR="$HOME/.claude/channels/telegram-<agentname>" claude --dangerously-skip-permissions --channels "plugin:telegram@claude-plugins-official"'

# Variants pra typo-tolerance
alias <Agentname>='<agentname>'
alias <AGENTNAME>='<agentname>'
```

Importante: usuário precisa **abrir terminal NOVO** depois (não funciona dentro de Claude Code já rodando — alias é do shell zsh).

### Fase 4: Telegram (preservar canal)

Veja [References/telegram-migration.md](References/telegram-migration.md). Resumo:

1. **Recupere o bot token** do OpenClaw (1Password ou `.env` no servidor)
2. **Confirme que o agente OpenClaw está desligado** — senão dois processos competem pelo mesmo bot
3. **Instale Bun**: `brew install oven-sh/bun/bun` — **CRÍTICO**, plugin não roda sem isso
4. **Crie state dir**: `~/.claude/channels/telegram-<agentname>/` com `.env` (chmod 600), `inbox/`
5. **Pré-aprove o sender ID do usuário** em `access.json` (skill `/telegram:access` tem bug com state dir custom — fazer manualmente):
   ```json
   {
     "dmPolicy": "allowlist",
     "allowFrom": ["<senderId>"],
     "groups": {},
     "pending": {}
   }
   ```
   E criar `approved/<senderId>` com chatId como conteúdo.
6. **Sender ID** descobre via Telegram API: `curl "https://api.telegram.org/bot<TOKEN>/getUpdates"` (depois do user mandar uma mensagem qualquer pro bot)
7. Instale o plugin local do Telegram dentro de uma sessão Claude Code:
   ```
   /plugin install telegram@claude-plugins-official
   ```

### Fase 5: Slack outbound (preservar identidade)

Veja [References/slack-migration.md](References/slack-migration.md). Resumo:

1. **Recupere o bot token** do OpenClaw (1Password ou `.env` no servidor)
2. **Valide token**: `curl -H "Authorization: Bearer <TOKEN>" https://slack.com/api/auth.test` — confirma bot user_id, team
3. **Salve credenciais** em `~/.claude/channels/slack-<agentname>/.env` (chmod 600)
4. **Crie helper CLI** em `~/.local/bin/<agentname>-slack-dm` (template em References)
5. **Documente a CLI** no CLAUDE.md do agente — incluir user IDs principais e canais relevantes que ele deve conhecer
6. **Teste**: `<agentname>-slack-dm <user_id> "🐝 test"` — usuário confirma que viu DM como o bot

⚠️ **Limitação:** Claude Code não tem plugin "channels" pra Slack (só Telegram). Slack é **outbound only** — pra usuário falar com o agente, usa Telegram ou terminal local.

### Fase 6: Cron jobs / heartbeat (preservar schedules)

Veja [References/launchd-cron-migration.md](References/launchd-cron-migration.md). Resumo:

1. **Identifique cada task no HEARTBEAT.md** do canônico — cada uma tem cron + prompt específico
2. **Crie estrutura:**
   ```
   ~/.claude/scripts/
     <agent>-task-runner.sh         # runner genérico
     <agent>-tasks/
       <task1>.md                    # prompt da task
       <task2>.md
       ...
   ~/Library/LaunchAgents/
     <org>.<agent>.<task1>.plist
     <org>.<agent>.<task2>.plist
     ...
   ```
3. **Runner script** lê o prompt do `.md`, executa `claude --print --dangerously-skip-permissions "<prompt>"` no diretório do agente, loga em `~/.claude/logs/<agent>-<task>.log`
4. **Plist usa StartCalendarInterval** com horário **local** (não UTC). Para M-F, 5 entries (Weekday 1-5).
5. **PATH no script** precisa incluir `~/.local/bin`, `/opt/homebrew/bin` (launchd usa PATH minimal)
6. **Carrega**: `launchctl bootstrap gui/$(id -u) <plist>`
7. **Testa**: `launchctl kickstart -k gui/$(id -u)/<label>` (dispara agora, sem esperar cron)

**Tasks que dependem de browser** (BUFIN, SumSub cockpit, etc.) não rodam headless — `claude --print` não tem GUI. Trate como manual ou use computer-use API se disponível.

### Fase 7: Validação ponta-a-ponta

Antes de desligar OpenClaw, **valide tudo no Claude Code**:

- [ ] Lança terminal: `<agentname>-terminal` — agente acorda, lê BOOTSTRAP, se apresenta com personalidade
- [ ] Lança com Telegram: `<agentname>` — agente conecta no bot, responde DM no Telegram
- [ ] Slack outbound: agente manda DM ou post via `<agentname>-slack-dm`, usuário vê DM como o bot
- [ ] Heartbeat manual: `launchctl kickstart -k gui/$(id -u)/<heartbeat label>` — fire imediato, log mostra execução completa, DM Slack ou Telegram entregue
- [ ] Daily log: `<agent_dir>/memory/YYYY-MM-DD.md` foi criado/atualizado

Só avançar pra Fase 8 depois que **TUDO** acima funcionar.

### Fase 8: Shutdown OpenClaw (preserva rollback)

Veja [References/openclaw-shutdown.md](References/openclaw-shutdown.md). Resumo:

1. **Backup do `openclaw.json`** antes de mexer
2. **Desativa o agente** na config (set `enabled: false`, NÃO delete)
3. **Mata processo** se estiver rodando: `kill <pid>` (sem -9 primeiro)
4. **Verifica webhook Telegram** está limpo (não deve ter URL apontando pra OpenClaw)
5. **Preserva workspace** dele (`~/.openclaw/workspace/agents/<name>/` continua intacto)
6. **Documenta rollback** num memory note (comando exato pra re-ativar se Claude Code não servir)

### Fase 9: Memória + handoff

Salve memórias de longo prazo:

- `reference_<agent>_telegram_bot.md` — bot ID, token location, state dir
- `reference_<agent>_slack_bot.md` — bot user_id, team, CLI path
- `reference_<agent>_launchd_jobs.md` — lista de plists, comandos úteis
- `project_<agent>_migration.md` — data, motivo (se memorável), rollback path

E avise outros agentes/usuários relevantes:
- DM no Slack pro owner se você refatorou vault root
- Atualiza Skynet docs se houver índice central

## Referências detalhadas

- [Inventário do agente OpenClaw](References/inventory-openclaw.md)
- [Isolamento de vault multi-agente](References/vault-isolation.md)
- [Migração Telegram (preservar bot)](References/telegram-migration.md)
- [Migração Slack outbound (preservar identidade)](References/slack-migration.md)
- [Migração de cron jobs via launchd](References/launchd-cron-migration.md)
- [Shutdown OpenClaw (preservar rollback)](References/openclaw-shutdown.md)

## Gotchas conhecidos

| Sintoma | Causa | Fix |
|---|---|---|
| Plugin Telegram não polla mensagens | Bun não instalado | `brew install oven-sh/bun/bun` |
| Pareamento Telegram falha | `/telegram:access` ignora `TELEGRAM_STATE_DIR` | Aprovação manual via `access.json` |
| Beyoncé/agente herda identidade do Dr. Sócrates (ou outro) | CLAUDE.md no root do vault contamina subpastas | Refatorar pra cada agente em dir isolado |
| `claude --print` headless não acha `claude` ou `bun` | launchd usa PATH minimal | Exportar PATH no script |
| /schedule remoto sem Slack/Linear | Connectors não conectados em claude.ai | Conectar OU usar launchd local |
| Bot token vaza em config remoto | Embeded em prompt de routine | Usar launchd local com `.env`, ou autorizar explicitamente |
| Dois processos competem pelo mesmo bot | OpenClaw ainda rodando + Claude Code novo | Desligar OpenClaw antes de subir Claude Code |

## Princípios

- **Preserve identidade.** Não recrie do zero — aponte pros canônicos.
- **Preserve canais.** Mesmo bot, mesma conversa, mesmo histórico.
- **Preserve schedules.** Mesmos horários, mesmos canais de saída.
- **Rollback existe.** OpenClaw não é deletado, é desativado.
- **Valide antes de desligar.** Nada de OpenClaw off com Claude Code não-testado.
- **Coordene em vault compartilhado.** Avisa outros owners antes de mexer no root.
