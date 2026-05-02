# Overclock

**Skill de orquestração multi-agente para o Claude Code.** O Overclock transforma uma sessão única do Claude num grid de painéis visíveis onde podes lançar agentes, delegar trabalho, executar modelos em paralelo e coordenar esquadrões especialistas — tudo sem sair do teu ambiente de desenvolvimento.

---

## O que é o Overclock?

O Overclock é uma camada de IDE sobre o Claude Code que expõe **7 ferramentas MCP** para orquestrar múltiplos agentes de IA em simultâneo. Cada sessão do Claude é um painel visível num grid de trabalho. Com ele podes:

- **Lançar novos painéis** e enviar-lhes tarefas independentes
- **Executar múltiplos modelos em paralelo** (Claude, Gemini, Codex, MIMO)
- **Coordenar esquadrões especialistas** com nomes próprios (marca, copy, cibersegurança, design, e mais)
- **Acompanhar projetos com múltiplos marcos** através de uma lista de tarefas visível em tempo real

A ideia central: se uma tarefa é paralelizável, vai bloquear o painel atual por mais de 5 minutos, ou beneficia de modelos cognitivos diferentes em subtarefas distintas — o Overclock torna essa orquestração explícita, visível e controlável.

---

## Estrutura do Repositório

```
overclock/
├── SKILL.md                   # Skill principal — árvore de decisão e padrões base
└── references/
    ├── tools.md               # Referência completa das 7 ferramentas do Overclock
    ├── recipes.md             # Exemplos práticos de orquestração do início ao fim
    ├── providers.md           # Guia de seleção de providers e modelos com matriz de trade-offs
    └── squads.md              # 8 esquadrões especialistas e padrões de ativação
```

`SKILL.md` é o ponto de entrada para o Claude Code. Os ficheiros de referência são carregados conforme necessário.

---

## As 7 Ferramentas

Todas as ferramentas têm o prefixo `mcp__overclock__` no registo de ferramentas.

| Ferramenta | Propósito | Quando usar |
|------------|-----------|-------------|
| `pane_list` | Descobrir painéis existentes e o seu estado | Antes de lançar — reutiliza painéis inativos sempre que possível |
| `pane_list_providers` | Listar providers de LLM configurados e os seus modelos | Antes de lançar qualquer painel com provider não predefinido |
| `pane_spawn` | Abrir um novo painel no grid de trabalho | Quando tens uma subtarefa paralelizável |
| `pane_write` | Enviar um prompt para um painel como se o tivesses digitado | Após lançar (ou para escrever num painel existente) |
| `pane_wait_idle` | Bloquear até o painel terminar | Sempre antes de ler — nunca leias um painel a correr |
| `pane_read` | Obter as últimas N linhas do output de um painel | Após `pane_wait_idle` confirmar que o painel terminou |
| `todo_manager` | Criar e atualizar uma lista de tarefas visível em tempo real | Projetos com 3 ou mais marcos distintos a nível de milestone |

Assinaturas completas, formatos de retorno e casos limite estão documentados em [`references/tools.md`](references/tools.md).

---

## Decisão Central: Quando Orquestrar

A maioria dos pedidos não precisa de orquestração. Por defeito, resolve tudo inline. Só recorre às ferramentas do Overclock quando a tarefa satisfaz um destes critérios:

```
O trabalho é paralelizável?                  → lança painéis
Precisa de um modelo diferente?              → lança painel com esse modelo
Tem 3+ marcos distintos?                     → todo_manager
Vai bloquear este painel por mais de 5 min?  → lança painel, devolve controlo ao utilizador
O utilizador disse "em paralelo"?            → lança painéis, sem mais deliberação
Nenhuma das anteriores?                      → faz a tarefa inline
```

Lançar um painel custa ~5–10s de tempo de arranque, e o subpainel não tem memória da conversa atual. Se uma tarefa demora 2 minutos inline, delegar é mais lento.

---

## Os Três Padrões Principais

### Padrão A: Delegar ao Painel de Trabalho

Cada sessão de orquestrador tem um painel de trabalho pré-atribuído. O seu ID aparece no system prompt em `squadOrchestratorWorkerId`. Usa-o para trabalho de execução que bloquearia o orquestrador.

```
pane_write(paneId: <worker-id>, text: "<brief autossuficiente da tarefa>")
pane_wait_idle(paneId: <worker-id>, timeoutMs: 300000)
pane_read(paneId: <worker-id>, lastN: 200)
```

O worker não tem contexto da tua sessão. Faz o brief como se fosse um colega que acabou de entrar na sala: objetivo, ficheiros de entrada, output esperado, onde guardar, restrições.

**Verifica se o worker já existe antes de lançar um novo painel.** Chama `pane_list()` primeiro — o objeto do painel orquestrador tem um campo `squadOrchestratorWorkerId` que aponta diretamente para o painel certo.

---

### Padrão B: Fan-Out em Paralelo

Quando o trabalho se divide em N subtarefas independentes, lança N painéis e escreve para todos eles na **mesma turn**. Escrever sequencialmente (um painel por turn) elimina o paralelismo.

```
# Turn 1 — lançar e escrever na mesma turn (execução concorrente começa imediatamente)
a = pane_spawn(model: "claude-sonnet-4-6")
b = pane_spawn(model: "claude-sonnet-4-6")
pane_write(a, "<brief da tarefa A>")
pane_write(b, "<brief da tarefa B>")

# Turn 2 — esperar por ambos
pane_wait_idle(a, timeoutMs: 300000)
pane_wait_idle(b, timeoutMs: 300000)

# Turn 3 — ler ambos
pane_read(a, lastN: 200)
pane_read(b, lastN: 200)
```

Para outputs muito grandes, pede aos subpainéis que guardem em ficheiros e lê esses ficheiros com a ferramenta `Read` padrão — o buffer do painel tem limite de ~1000 linhas.

---

### Padrão C: Esquadrão Multi-Modelo

Modelos diferentes para trabalhos cognitivos diferentes na mesma tarefa. Por exemplo: Opus para crítica arquitetural (lento, raciocínio profundo), Sonnet para implementação (rápido, orientado à execução), Gemini para dados frescos da web.

**Chama sempre `pane_list_providers()` primeiro.** Os IDs dos providers são locais à máquina — variam entre instalações. Passar um ID desconhecido dá erro.

```
providers = pane_list_providers()
# verifica que gemini-cli, codex-cli, etc. estão presentes e anota os IDs exatos

opus    = pane_spawn(model: "claude-opus-4-7")
gemini  = pane_spawn(providerId: "gemini-cli", model: "gemini-2.5-flash")
mimo    = pane_spawn(providerId: "mimo-FxzXvc", model: "mimo-v2.5-pro")
```

---

## Providers e Seleção de Modelos

Esta instalação inclui quatro providers. Os IDs abaixo são específicos desta máquina — verifica sempre com `pane_list_providers()` em tempo de execução.

| Provider ID | Label | Modelos |
|-------------|-------|---------|
| `claude-oauth` | Claude Code (predefinido) | `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5` |
| `gemini-cli` | Gemini CLI | `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-3-flash-preview` |
| `codex-cli` | Codex CLI | `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.2` |
| `mimo-FxzXvc` | MIMO Token Plan SGP | `mimo-v2.5-pro`, `mimo-v2.5`, `mimo-v2-pro`, `mimo-v2-omni` |

### Quando usar cada modelo

| Tarefa | Modelo recomendado |
|--------|--------------------|
| Decisões de arquitetura, revisão de segurança, raciocínio complexo | **Claude Opus 4.7** |
| Implementação de funcionalidades, refactors multi-ficheiro, testes | **Claude Sonnet 4.6** |
| Escrita de documentação, transformações mecânicas, análise de logs | **Claude Haiku 4.5** |
| Pesquisa na web, informação recente, segunda opinião estilística | **Gemini 2.5 Flash** |
| Código algorítmico, terceira perspetiva em brainstorms | **Codex GPT-5.4** |
| Trabalho em massa com orçamento de tokens limitado | **MIMO Pro** |
| Brainstorm multi-perspetiva | Opus + Gemini + Codex em conjunto |

**Erros comuns:**
- Não uses Opus para tudo "por precaução" — é lento e caro. O Sonnet faz a maioria do trabalho de execução igualmente bem.
- Não uses Haiku para tarefas ambíguas — perde nuance em raciocínio complexo.
- Não trates o MIMO como substituto direto do Claude em trabalho crítico — é compatível mas não idêntico.

Guia completo em [`references/providers.md`](references/providers.md).

---

## todo_manager: Acompanhamento de Marcos

Usa `todo_manager` quando um projeto tem **3 ou mais tarefas distintas a nível de milestone**. Não o uses para builds simples, correções de bugs ou perguntas conversacionais — é overhead desnecessário para pedidos triviais.

```
# Configura a lista de tarefas visível (máx. 7 tarefas; a primeira fica ativa imediatamente)
todo_manager(action: "set_tasks", tasks: [
  "Atualizar schema da base de dados",
  "Refatorar rotas da API",
  "Atualizar handlers de webhooks",
  "Atualizar frontend",
  "Executar testes end-to-end"
])

# Avança a lista assim que cada marco termina — chama imediatamente, não acumules
todo_manager(action: "move_to_task", moveToTask: "Refatorar rotas da API")

# Sinaliza o fim do projeto
todo_manager(action: "mark_all_done")
```

O progresso em tempo real é a funcionalidade. Se acumulares as chamadas a `move_to_task` e as fizeres todas no final, o utilizador não vê nada atualizar até ao último momento.

**Usa granularidade de milestone.** "Integrar formulário de registo" lê-se como progresso. "Adicionar import statement" lê-se como ruído.

---

## Esquadrões de Agentes

Esta instalação inclui 8 esquadrões especialistas com ~100 agentes pré-configurados no total. Cada esquadrão tem um chefe que faz triagem e encaminha para o especialista certo.

| Esquadrão | Chefe | Agentes | Domínio |
|-----------|-------|---------|---------|
| advisory-board | `@board-chair` | 11 | Pensamento estratégico, decisões executivas |
| brand-squad | `@brand-chief` | 15 | Estratégia e identidade de marca |
| claude-code-mastery | `@claude-mastery-chief` | 8 | Claude Code: hooks, skills, MCP, equipas de agentes |
| copy-squad | `@copy-chief` | 23 | Copywriting — 22 copywriters lendários |
| cybersecurity | `@cyber-chief` | 15 | Operações de segurança ofensiva e defensiva |
| data-squad | `@data-chief` | 7 | Analytics, CLV, crescimento, audiências |
| design-squad | `@design-chief` | 8 | Operações de design e UX |
| hormozi-squad | `@hormozi-chief` | 16 | Frameworks de escala de negócio de Alex Hormozi |

### Padrão de ativação

```
@<esquadrão>-chief              # ativa o chefe — inicia triagem
*diagnose                       # pede ao chefe para fazer triagem do teu problema
*<nome-do-workflow>             # executa um workflow nomeado do esquadrão
@<esquadrão>-chief:<agente>     # fala diretamente com um agente específico
```

Exemplos:
```
@brand-chief                          # ativa o esquadrão de marca
*brand-creation                       # executa o workflow completo de criação de marca
@copy-chief:direct-response-writer    # fala com o especialista em direct response
```

### Esquadrão vs. painel individual

| Situação | Usa |
|----------|-----|
| O domínio corresponde a um dos 8 esquadrões | Ativa o chefe do esquadrão |
| Execução técnica (construir funcionalidade, corrigir bug) | Lança um painel normal com `pane_spawn` |
| Problema cross-domínio | Painel individual ou vários chefes em paralelo |

Os esquadrões são estruturas de conhecimento — moldam a forma como o Claude responde com expertise de domínio, frameworks e workflows. Os painéis lançados são capacidade de execução — fazem trabalho em paralelo. São ortogonais: podes lançar um painel e ativar um esquadrão dentro dele.

### Combinar esquadrões com painéis

```
brand_pane = pane_spawn(model: "claude-opus-4-7")
copy_pane  = pane_spawn(model: "claude-sonnet-4-6")

pane_write(brand_pane, "@brand-chief\n*diagnose\nEstamos a lançar o SwipeScale para equipas de vendas B2B...")
pane_write(copy_pane,  "@copy-chief\n*diagnose\nPrecisamos de copy para a landing page do SwipeScale...")
```

Cada painel executa o seu esquadrão de forma independente. O orquestrador recolhe os outputs e sintetiza.

Referência completa dos esquadrões em [`references/squads.md`](references/squads.md).

---

## Receitas

Quatro exemplos práticos do início ao fim estão documentados em [`references/recipes.md`](references/recipes.md):

| Receita | Quando usar |
|---------|-------------|
| **Esquadrão de code review com dois painéis** | Crítica (Opus) + remediação (Sonnet) em paralelo |
| **Documentação + implementação em paralelo** | Ambos dependem da mesma spec, nenhum bloqueia o outro |
| **Brainstorm multi-perspetiva** | Mesmo prompt em Opus, Gemini e Codex para divergência genuína |
| **Migração longa com todo_manager** | Marcos sequenciais, worker faz o trabalho pesado, orquestrador narra o progresso |

---

## Erros Comuns

1. **Ler antes de esperar** — `pane_read` num painel a correr devolve output parcial. Chama sempre `pane_wait_idle` primeiro.
2. **Fan-out sequencial** — múltiplas chamadas a `pane_write` têm de acontecer na mesma turn, não uma por turn. Writes sequenciais eliminam o paralelismo.
3. **Briefs vagos para subpainéis** — os subpainéis não têm contexto da conversa atual. Especifica o objetivo, ficheiros de entrada, formato do output esperado e onde guardar.
4. **Lançar quando um worker já existe** — chama `pane_list()` primeiro. Reutiliza o painel de trabalho inativo antes de lançar um novo.
5. **IDs de providers hardcoded** — IDs como `mimo-FxzXvc` são locais à máquina e serão diferentes noutras instalações. Chama sempre `pane_list_providers()` em tempo de execução.
6. **Todos em micro-passos** — `todo_manager` é para 3–7 deliverables a nível de milestone, não para cada ação individual.
7. **Usar todo_manager para pedidos triviais** — para uma tarefa simples, a lista de tarefas é overhead desnecessário.
8. **Acumular chamadas a `move_to_task`** — chama imediatamente quando cada milestone termina. O utilizador vê a lista atualizar em tempo real.

---

## Quando NÃO Orquestrar

- Perguntas de conhecimento puro ("O que é X?", "Explica Y") — responde inline
- Correções de bugs, refactors, edições de ficheiro único — sem benefício de paralelismo
- Qualquer coisa que uma resposta inline de dois minutos resolve — delegar seria mais lento

Em caso de dúvida, faz a tarefa diretamente. A orquestração é uma ferramenta para formas específicas de trabalho, não um modo predefinido.

---

## Aviso sobre o Painel de Trabalho

Podes ser o orquestrador, ou podes ser um painel de trabalho para o qual outro orquestrador está a delegar. Se o teu prompt inicial parece uma tarefa autossuficiente sem enquadramento de orquestrador no system prompt, és um worker — executa a tarefa diretamente sem lançar mais painéis.
