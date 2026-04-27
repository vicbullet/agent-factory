# Isolamento de vault multi-agente

Quando o vault (Obsidian + Google Drive) tem **`.claude/CLAUDE.md` no root**, qualquer agente que rodar em **qualquer subpasta** desse vault herda essa identidade. Bug clássico: agente A com identidade do agente B porque B configurou o root primeiro.

## Sintoma

Um agente recém-criado em `<Vault>/04 Skynet/Agents/<NewAgent>/` boota e se identifica como o **owner antigo do vault** (ex: Dr. Sócrates), porque Claude Code carrega CLAUDE.md de cima pra baixo (raiz → cwd) e mergeia tudo.

## Causa

Claude Code resolve CLAUDE.md hierarquicamente:
1. `~/CLAUDE.md` (user-level)
2. `<Vault>/.claude/CLAUDE.md` (project-level — se existe)
3. `<Vault>/<subpath>/CLAUDE.md` (cwd-level)

Tudo é mergeado. O CLAUDE.md do root vence quando tem identidade.

## Solução

**Cada agente vive em diretório isolado, com seu próprio CLAUDE.md.** O root do vault só tem **contexto compartilhado** (regras de segurança, convenções, sem identidade).

### Estrutura recomendada

```
<Vault>/
├── .claude/
│   └── CLAUDE.md              # ← contexto compartilhado neutro (security, convenções)
└── 04 Skynet/                 # ou onde os agentes vivem
    └── Agents/
        ├── <Agent1>/
        │   └── CLAUDE.md      # ← identidade do Agent1
        ├── <Agent2>/
        │   └── CLAUDE.md      # ← identidade do Agent2
        └── ...
```

### Refatoração (quando o root já tem identidade)

Se o root atualmente tem `<Vault>/.claude/CLAUDE.md` com identidade de algum agente (ex: Dr. Sócrates do Pedro), você precisa:

#### Passo 1: Mover identidade do root pra dir dedicada

1. Ler conteúdo de `<Vault>/.claude/CLAUDE.md`
2. Criar `<Vault>/04 Skynet/Agents/<OriginalAgent>/CLAUDE.md` com o conteúdo (acrescenta uma nota sobre o novo launch dir)
3. Esvaziar / refatorar `<Vault>/.claude/CLAUDE.md` pra contexto compartilhado neutro

**Template do CLAUDE.md compartilhado neutro:**

```markdown
# <Vault Name> — Shared Context

> Contexto compartilhado deste vault. Todo agente que rodar a partir de qualquer subpasta herda essas regras de segurança e contexto. Nenhuma identidade de agente vive aqui.

---

## Identidade dos agentes

Cada agente vive em `<path>/<AgentName>/CLAUDE.md` e é lançado a partir do próprio diretório.

Agentes ativos:
- **<Agent1>** — `<path>/<Agent1>/` — <papel> (<owner>)
- **<Agent2>** — `<path>/<Agent2>/` — <papel> (<owner>)

---

## Security Rules (compartilhadas)

- Trabalhar dentro deste vault para todas as operações <Org>
- NUNCA acessar `~/Library/` exceto este Google Drive (se aplicável)
- NUNCA deletar arquivos do vault sem confirmação explícita do owner do agente
- NUNCA compartilhar informações não relacionadas à <Org>
- NUNCA expor tokens/API keys em outputs
- Preferir `trash` sobre `rm`

---

## Convenções do vault

[lista de convenções específicas: estrutura de pastas, frontmatter, wikilinks, etc.]

---

## [Outras seções compartilhadas: times Linear conhecidos, integrações standard, etc.]
```

#### Passo 2: Atualizar alias do owner original

O owner do agente que estava no root precisa atualizar o alias dele pra lançar do diretório dedicado:

```bash
# Antes
alias <agentname>='cd "<Vault>" && claude'

# Depois
alias <agentname>='cd "<Vault>/<path>/<AgentName>" && claude'
```

#### Passo 3: Avisar o owner

⚠️ **Vault compartilhado = mudança visível pra todos.** Se você não é o único usuário do vault, **comunique a mudança** antes ou imediatamente depois.

DM no Slack pro owner explicando:
- O que mudou (path da identidade dele)
- Por quê (suportar multi-agente)
- O que ele precisa fazer (atualizar alias)
- Que tudo (regras, API keys, heartbeat) foi preservado, só mudou de endereço

Template de mensagem:

```
Fala, <Owner>. Heads-up: refatorei o `<Vault>/.claude/CLAUDE.md` pra suportar multi-agente.

**O que mudou:**
- Identidade do <Original Agent> foi movida pra `<path>/<Original Agent>/CLAUDE.md`
- O `.claude/CLAUDE.md` do root agora tem só contexto compartilhado (security, convenções)
- Criei o <New Agent> em `<path>/<New Agent>/`

**Por quê:** com a identidade do <Original> no root, qualquer agente que rodasse em qualquer subpasta herdava ele.

**O que muda pra você:** o alias do <Original> precisa lançar de `<path>/<Original>/` em vez do root:

`alias <name>='cd "<full path>" && claude --dangerously-skip-permissions'`

API keys e regras tão preservadas, só mudaram de endereço.

Qualquer coisa estranha, me avisa.
```

## Quando NÃO refatorar

- Se você é o único usuário do vault: refatora sem cerimônia
- Se o root nunca teve identidade (CLAUDE.md genérico ou ausente): só cria o novo agente, não precisa mexer no root
- Se o owner do root não responder e a mudança é urgente: refatora E manda mensagem (urgência justifica unilateralidade quando reversível)

## Quando colocar agente FORA do vault

Se o vault é shared e o usuário não pode coordenar refatoração agora:

```bash
# Move agente pra dir local fora do vault
mkdir -p ~/<org>-agents/<agentname>
# ... cria arquivos lá ...
```

Trade-off: perde a integração visual com Obsidian (vault não vê o agente na hierarquia). Mas ganha isolamento total.

## Validação

Depois de refatorar, valide:

1. Lança o agente novo: `<newagent>-terminal`
2. Pergunta: "Quem é você?"
3. Resposta deve ser do agente novo, **sem mencionar identidade alheia**
4. Se ele mencionar Dr. Sócrates ou outro: refatoração não pegou — verifica os CLAUDE.md
