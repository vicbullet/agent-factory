# Shutdown OpenClaw (preservar rollback)

Desativa o agente no OpenClaw **sem deletar nada**, pra preservar a opção de rollback caso a versão Claude Code não atenda.

⚠️ **Sequenciamento crítico:** só desliga o OpenClaw DEPOIS de validar 100% no Claude Code (Fase 7 da skill). Nunca desliga antes.

## Pré-requisitos

- [ ] Validação ponta-a-ponta no Claude Code passou (Telegram + Slack outbound + ao menos 1 cron disparado com sucesso)
- [ ] Inventário do OpenClaw concluído (sabe onde estão arquivos, processos, config)
- [ ] Acesso SSH/Cloudflared ao servidor OpenClaw

## Estratégia: dois caminhos pra rodar comandos no servidor

### Caminho A (recomendado): Claude Code do servidor

Cola o prompt abaixo numa sessão Claude Code rodando **dentro do servidor OpenClaw**:

```
Preciso desativar o agente <AgentName> do OpenClaw — vou usar a versão Claude Code local.
Estratégia: desativar limpo e reversível, sem deletar nada.

Faz nesta ordem e me reporta cada passo:

1. **Inspecionar config atual**
   - cat ~/.openclaw/openclaw.json | grep -A 20 "<agentname>"
   - Identifica onde mora o workspace dela (provável: ~/.openclaw/workspace/agents/<agentname>/)

2. **Backup da config antes de mexer**
   - cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-$(date +%Y%m%d-%H%M%S)

3. **Desativar o agente na config**
   - Edita ~/.openclaw/openclaw.json mudando o status do agente. Semântica depende do schema:
     - Se tem `enabled: true` → muda pra `false`
     - Se tem `status: "active"` → muda pra `"disabled"`
     - Se tem array de agentes ativos → remove o nome do array (mantém objeto config dele em outro lugar pra rollback)
   - **NÃO deletes** a entry inteira nem o workspace — só desativa
   - Mostra o diff antes/depois

4. **Matar processo rodando (se houver)**
   - ps aux | grep -i <agentname> | grep -v grep
   - Se aparecer processo: kill <PID> (sem -9 primeiro; só -9 se resistir)
   - Confirma: ps aux | grep -i <agentname> | grep -v grep deve retornar vazio

5. **Recarregar OpenClaw (se necessário)**
   - Se OpenClaw tem daemon/scheduler que precisa de reload pra pegar a config nova, executa o reload. Se não souber qual o comando, me pergunta antes de chutar.

6. **Verificar webhook Telegram desconectado**
   - curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
   - Se webhook tiver URL apontando pra OpenClaw, deleta: curl -s "https://api.telegram.org/bot<TOKEN>/deleteWebhook"

7. **Reportar pra mim:**
   - ✅ Agente desativado na config (com diff)
   - ✅ Processo morto (ou já não estava rodando)
   - ✅ Webhook deletado (ou não tinha)
   - 📁 Caminho do workspace dele preservado pra rollback
   - 🔄 Comando de rollback exato (ex.: cp <backup> ~/.openclaw/openclaw.json && reload)

NÃO deletes nada do workspace dele. NÃO faça /revoke no BotFather. Se algo cheirar mal (config com schema desconhecido, processo zumbi, webhook estranho), para e me pergunta antes de prosseguir.
```

⚠️ Substitua `<AgentName>` e `<agentname>` por o nome real (formato apropriado), e `<TOKEN>` pelo bot token do Telegram.

### Caminho B: SSH direto

Se Claude Code não está no servidor, usa SSH:

```bash
# Establish connection (browser auth via cloudflared se aplicável)
cloudflared access ssh-gen --hostname <openclaw-host>

# Backup
ssh -o ProxyCommand="cloudflared access ssh --hostname %h" \
    -o IdentityFile=~/.cloudflared/<host>-cf_key \
    <user>@<host> \
    'cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-$(date +%Y%m%d-%H%M%S)'

# Edita config (manual via vi/nano remoto)
ssh ... 'vi ~/.openclaw/openclaw.json'

# Kill processo
ssh ... 'pkill -f <agentname> || true'

# Verifica webhook
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

Mais propenso a erro humano. Prefere caminho A.

## O que NÃO fazer

- ❌ **Deletar workspace** (`rm -rf ~/.openclaw/workspace/agents/<name>/`) — perde rollback
- ❌ **Remover entry inteira** do `openclaw.json` sem backup — schema diff fica difícil de reverter
- ❌ **`/revoke` no BotFather** — invalida token usado pelo Claude Code também
- ❌ **`kill -9` first** — pode deixar estado inconsistente em arquivos abertos
- ❌ **Mexer em outros agentes** — só desativa o que está migrando

## Documentar rollback

Salva memória pra ti mesmo (auto-memory) e/ou nota no Slite:

```markdown
# Rollback <AgentName> OpenClaw

Data desativação: <YYYY-MM-DD HH:MM>

## Estado preservado

- Config backup: ~/.openclaw/openclaw.json.bak-<timestamp>
- Workspace intacto: ~/.openclaw/workspace/agents/<agentname>/
- Tokens (Telegram, Slack) ainda válidos no `.env` do workspace

## Comando de rollback (re-ativar OpenClaw)

```bash
ssh <user>@<openclaw-host> 'cp ~/.openclaw/openclaw.json.bak-<timestamp> ~/.openclaw/openclaw.json && <reload command>'
```

## Pré-requisito de rollback

Antes de re-ativar OpenClaw:
1. Desativar/desligar plists launchd local: `launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/<org>.<agent>.*.plist`
2. Sair de qualquer sessão `<agent>` local
3. Confirmar que ninguém vai rodar dois processos no mesmo bot Telegram
```

## Quando re-ativar OpenClaw

Cenários onde o rollback faz sentido:

- Claude Code local tá inadequado (Mac dormindo bloqueia heartbeat críticos)
- Performance ruim no headless (timeouts, MCPs flakey)
- Algum recurso só existe no OpenClaw (ex: orquestração multi-agente)

Cenários onde NÃO precisa rollback (resolve no Claude Code):

- Falta um cron job → adiciona plist
- Bot não responde → debug local (Bun, alias, plugin)
- Personalidade fraca → refina SOUL.md no canônico

## Pós-shutdown: limpeza eventual

Depois de **30+ dias** de Claude Code estável (sem rollback necessário), você pode:

1. **Arquivar workspace** (não deletar): `mv ~/.openclaw/workspace/agents/<name> ~/.openclaw/workspace/_archive/<name>-<date>`
2. **Remover entry definitiva da config** (manter no backup)
3. **Documentar deprecation** no Slite/Skynet docs

Mas mantém os backups e o token até ter certeza absoluta. Token revogado é caro de re-gerar (precisa atualizar tudo que dependia dele).

## Sinais que o shutdown deu certo

- [ ] OpenClaw config tem o agente desativado (diff confirmado)
- [ ] `ps aux | grep <agentname>` retorna vazio no servidor
- [ ] `getWebhookInfo` retorna `url: ""` (sem webhook OpenClaw)
- [ ] Mensagem ao bot Telegram só é respondida pelo Claude Code local (não duplicada por OpenClaw)
- [ ] DM Slack do bot só vem do Claude Code (não duplicada)
- [ ] Crons OpenClaw param de aparecer nos logs deles
- [ ] Nenhum erro novo em monitoramento OpenClaw
- [ ] Backup arquivado e documentado
