# Migração de cron jobs via launchd

Reproduz os schedules do `HEARTBEAT.md` no Claude Code via `launchd` (cron do macOS). Cada task vira um agente do launchd que dispara `claude --print` headless no diretório do agente.

## Vantagens vs `/schedule` remoto

| Critério | launchd local | /schedule remoto |
|---|---|---|
| **24/7** | ❌ só com Mac ligado | ✅ |
| **Acesso a CLIs locais** (`<agent>-slack-dm`, etc.) | ✅ | ❌ |
| **Acesso a arquivos canônicos do agente** | ✅ | ❌ (precisa repo git) |
| **MCPs** | ✅ todos os configurados localmente | ❌ só os de claude.ai/customize/connectors |
| **Browser tasks** (BUFIN, SumSub portal) | possível com computer-use | ❌ |
| **Tokens em config** | ✅ via `.env` local seguro | ⚠️ vazam pra prompt do routine |

**Default desta skill: launchd local.** Use /schedule remoto só pra tasks MCP-only que precisam rodar quando Mac está off.

## Estrutura

```
~/.claude/scripts/
├── <agent>-task-runner.sh        # runner genérico
└── <agent>-tasks/
    ├── <task1>.md                 # prompt da task1
    ├── <task2>.md
    └── ...

~/.claude/logs/                   # logs de cada task
├── <agent>-<task1>.log
├── <agent>-<task1>.launchd.log   # log do launchd (stdout/stderr nível processo)
└── ...

~/Library/LaunchAgents/
├── <org>.<agent>.<task1>.plist
├── <org>.<agent>.<task2>.plist
└── ...
```

## Runner genérico

`~/.claude/scripts/<agent>-task-runner.sh`:

```bash
#!/usr/bin/env bash
# <agent>-task-runner.sh — runner genérico de tasks da <agent>
# Lê prompt de ~/.claude/scripts/<agent>-tasks/<task>.md e dispara claude --print headless.

set -euo pipefail

if [[ $# -lt 1 ]]; then
  echo "ERRO: uso: $0 <task-name>" >&2
  exit 1
fi

TASK="$1"
TASK_DIR="$HOME/.claude/scripts/<agent>-tasks"
TASK_FILE="$TASK_DIR/${TASK}.md"

[[ ! -f "$TASK_FILE" ]] && { echo "ERR: task file não existe: $TASK_FILE" >&2; exit 2; }

AGENT_DIR="<absolute path do agent dir>"
LOG_DIR="$HOME/.claude/logs"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/<agent>-${TASK}.log"

# launchd usa PATH minimal — restaurar tudo que o agente vai precisar
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
export HOME="$HOME"

cd "$AGENT_DIR"

TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S %Z")

{
  echo ""
  echo "════════════════════════════════════════════"
  echo "🐝 Task '$TASK' fired at $TIMESTAMP"
  echo "════════════════════════════════════════════"
  echo ""
  echo "--- <agent> output ---"
} >> "$LOG_FILE"

PROMPT=$(cat "$TASK_FILE")

claude --print --dangerously-skip-permissions "$PROMPT" >> "$LOG_FILE" 2>&1
EXIT_CODE=$?

echo "" >> "$LOG_FILE"
echo "--- Task '$TASK' finished at $(date '+%Y-%m-%d %H:%M:%S %Z') (exit $EXIT_CODE) ---" >> "$LOG_FILE"

exit $EXIT_CODE
```

⚠️ Substitua `<agent>` por o nome do agente (lowercase) e `<absolute path do agent dir>` pelo caminho real.

```bash
chmod +x ~/.claude/scripts/<agent>-task-runner.sh
mkdir -p ~/.claude/scripts/<agent>-tasks
```

## Prompt files

Pra cada task no `HEARTBEAT.md`, crie um `.md` em `<agent>-tasks/`:

```
~/.claude/scripts/<agent>-tasks/
├── heartbeat.md                       # se for usar runner pro heartbeat também
├── <task1>.md                          # ex: daily-action-items-briefing.md
├── <task2>.md                          # ex: merchant-email-followup.md
└── ...
```

**Cada prompt deve ser auto-suficiente** — o agente headless começa com zero contexto além do CLAUDE.md do diretório. Inclua:

1. **Identidade:** "Você é a <Agent>"
2. **Tarefa:** referência ao nome no `HEARTBEAT.md` original
3. **Objetivo:** o que precisa resultar
4. **Checklist:** passos concretos
5. **Entrega:** comando exato (`<agent>-slack-dm ...` ou similar)
6. **Formato:** template de saída (mrkdwn pra Slack, etc.)
7. **Regras:** silêncio fora de horário, threshold de escalação, anti-patterns
8. **Erro fatal:** como reportar sem dump técnico

Veja exemplo em [agent-home-template.md](agent-home-template.md) ou nos prompts canônicos do `HEARTBEAT.md`.

## Plist por task

`~/Library/LaunchAgents/<org>.<agent>.<task>.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string><org>.<agent>.<task></string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-lc</string>
        <string>/Users/<user>/.claude/scripts/<agent>-task-runner.sh <task></string>
    </array>

    <key>StartCalendarInterval</key>
    <!-- diário M-F: 5 entries -->
    <array>
        <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>HH</integer><key>Minute</key><integer>MM</integer></dict>
        <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>HH</integer><key>Minute</key><integer>MM</integer></dict>
        <dict><key>Weekday</key><integer>3</integer><key>Hour</key><integer>HH</integer><key>Minute</key><integer>MM</integer></dict>
        <dict><key>Weekday</key><integer>4</integer><key>Hour</key><integer>HH</integer><key>Minute</key><integer>MM</integer></dict>
        <dict><key>Weekday</key><integer>5</integer><key>Hour</key><integer>HH</integer><key>Minute</key><integer>MM</integer></dict>
    </array>
    <!-- OU semanal: dict simples -->
    <!--
    <dict>
        <key>Weekday</key><integer>1</integer>  // 1=Monday
        <key>Hour</key><integer>HH</integer>
        <key>Minute</key><integer>MM</integer>
    </dict>
    -->

    <key>RunAtLoad</key>
    <false/>

    <key>StandardOutPath</key>
    <string>/Users/<user>/.claude/logs/<agent>-<task>.launchd.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/<user>/.claude/logs/<agent>-<task>.launchd.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/<user></string>
    </dict>
</dict>
</plist>
```

### Regras importantes

- **Weekday:** 0=Sunday, 1=Monday, ..., 7=Sunday. M-F = 1-5.
- **Hour/Minute em horário LOCAL do Mac** (não UTC, ao contrário de cron expressions de /schedule).
- **Heartbeat 2h:** 25 entries (5 horas × 5 weekdays). Verbose mas é o preço.
- **`PATH` no shell** (`/bin/bash -lc`) — `-l` carrega `~/.zshrc` (se shell padrão for zsh) ou `~/.bash_profile`. Mas confiar nisso é frágil — explicite PATH no script (já está no template).

### Validar plist

```bash
plutil -lint ~/Library/LaunchAgents/<org>.<agent>.<task>.plist
# Output esperado: "<path>: OK"
```

## Carregar / descarregar / disparar / debug

```bash
# Carregar (uma vez por plist)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/<org>.<agent>.<task>.plist

# Status
launchctl print gui/$(id -u)/<org>.<agent>.<task> | head -30

# Disparar agora (sem esperar cron)
launchctl kickstart -k gui/$(id -u)/<org>.<agent>.<task>

# Tail log
tail -f ~/.claude/logs/<agent>-<task>.log

# Desativar (reversível)
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/<org>.<agent>.<task>.plist

# Listar todos jobs do agente
launchctl list | grep <agent>
```

## Test antes de bulk-load

Recomendação: depois de criar o **runner + 1 task + 1 plist**, dispara manualmente com `launchctl kickstart` antes de carregar os outros plists. Se o runner tiver bug, todos os crons falham.

```bash
# 1. carrega só uma task
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/<org>.<agent>.<task1>.plist

# 2. dispara
launchctl kickstart -k gui/$(id -u)/<org>.<agent>.<task1>

# 3. aguarda log
until grep -q "finished at" ~/.claude/logs/<agent>-<task1>.log; do sleep 5; done

# 4. valida resultado (DM Slack chegou? daily log atualizou? log mostra exit 0?)
tail -50 ~/.claude/logs/<agent>-<task1>.log
```

Se ok: bulk load os outros. Se não: debugga, corrige runner, redispara.

## Tasks que NÃO rodam headless

`claude --print` não tem GUI. Tasks que dependem de:

- **Browser** (BUFIN portal, SumSub cockpit, BS2 contestações, Atlassian via Chrome) — não funcionam direto.
- **Upload manual de arquivo** (P&L CSVs, deck templates) — manual ou semi-manual.

Opções:

1. **computer-use API** (se disponível) — agente controla Chrome via screenshot+click. Mais complexo, requer setup adicional.
2. **Skill local que faz scrape via API direto** — ex: BUFIN tem API REST? Linear/Atlassian via API em vez de browser?
3. **Manter manual** — não automatize o que precisa de humano. Anota como "on-demand" no CLAUDE.md do agente.

## Logs que crescem

Logs em `~/.claude/logs/` crescem indefinidamente. Adiciona rotação se incomodar:

```bash
# Adiciona ao ~/.zshrc ou cron
0 0 * * 0 find ~/.claude/logs -name "*.log" -size +10M -exec mv {} {}.old \;
```

Ou usa `newsyslog`/`logrotate`. Não crítico — investigue se chegar a 100+MB.

## Mac dormindo / off

`launchd` agente tipo `LaunchAgent` (não `LaunchDaemon`) só roda **quando o usuário está logado e o sistema está ativo**. Mac dormindo: cron NÃO dispara. Mac desligado: cron NÃO dispara.

Quando o Mac volta a ficar ativo, launchd **não re-dispara** os cron que perdeu (default). Comportamento OK pra agentes que não precisam fechar gap (heartbeat 2h: o próximo ciclo cobre o anterior).

Se precisar re-disparar perdidos: opção `<key>StartOnMount</key><true/>` ou `<key>RunAtLoad</key><true/>` (este último dispara TODA vez que carrega — geralmente não é o que quer).

Pra agentes que precisam 24/7 verdadeiro: usar /schedule remoto OU servidor sempre ligado.
