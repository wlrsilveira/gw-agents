# Fábrica de Software (Cursor Subagents)

Pipeline autônomo de entrega de features: da SDD ao PR aberto no remoto, sem colocar o usuário no meio do processo.

O usuário entra **duas vezes**: no início (entrega a SDD e diz "roda a fábrica"; e, se não houver branch `development`, indica a base) e no fim (recebe a URL do PR). Entre esses pontos, os especialistas se validam entre si e o `shipper` publica.

---

## Sumário

- [Como acionar](#como-acionar)
- [Os 8 agentes](#os-8-agentes)
- [O pipeline passo a passo](#o-pipeline-passo-a-passo)
- [Regras que governam a fábrica](#regras-que-governam-a-fábrica)
- [Pré-requisitos do projeto](#pré-requisitos-do-projeto)
- [Ciclos de correção e escalação](#ciclos-de-correção-e-escalação)
- [Uso avulso dos agentes](#uso-avulso-dos-agentes)
- [Limitações conhecidas](#limitações-conhecidas)

---

## Como acionar

Com uma SDD/spec pronta no projeto, qualquer uma destas frases dispara o pipeline completo:

- "faz aí"
- "roda a fábrica"
- "implementa a feature de `specs/xyz.md`"
- entregar uma SDD/spec direto no chat

O `orchestrator` assume, e você não é chamado de volta até o fim (ou até uma escalação legítima).

---

## Os 8 agentes

| Agente | Papel | Edita produção? | Quando roda |
|--------|-------|-----------------|-------------|
| `orchestrator` | Coordena o pipeline, monta os handoffs, roteia falhas | Não (salvo trivialidade sem dono) | Sempre — é a porta de entrada |
| `test-writer` | Escreve os testes **a partir da SDD, antes** do código | Só seams mínimas já padronizadas | Estágio 1 |
| `backend-laravel` | Implementa Laravel: domínio, serviços, API, Blade/Livewire/Inertia | Sim | Estágio 2 |
| `frontend` | Implementa React / Vue / Flutter em app separado | Sim | Estágio 2 (paralelo ao backend) |
| `verifier` | Roda os testes **da feature** + QA contra os aceites | Não | Estágio 3 |
| `regression` | Roda a suíte **completa** do sistema | Não | Estágio 4 |
| `code-reviewer` | Review final do diff contra SDD, arquitetura e segurança | Não (`readonly: true`) | Estágio 5 |
| `shipper` | Commit (sem co-autoria Cursor), push e abre o PR no remoto | Só git/PR | Estágio 6 |

---

## O pipeline passo a passo

### 0. Preparação + branch (`orchestrator`)

Antes de delegar qualquer coisa, o orquestrador confirma:

- Existe uma SDD/spec com **critérios de aceite testáveis** (`specs/…`, ticket, ou anexada no chat).
- O `AGENTS.md` do projeto documenta como rodar os testes de backend, frontend e regressão.

Se faltar critério observável ou comando de teste, ele **pergunta uma única vez** e para. Não adivinha comportamento de produto.

Em seguida configura o git:

1. Se existir branch `development` (local ou `origin`) → checkout nela (e atualiza do remoto quando fizer sentido).
2. Se **não** existir → **pergunta qual é a branch base** e só continua após a resposta.
3. Cria e faz checkout da branch com o **slug da spec** (ex. `specs/T001-scaffold-laravel.md` → branch `T001-scaffold-laravel`).
4. Deriva o **título do PR** no padrão `[T-001] Scaffold Laravel` (código com hífen + Title Case do resto do slug) e guarda base + branch + título para o `shipper`.

### 1. `test-writer` — os testes viram a fonte da verdade

Escreve os testes **antes** da implementação existir (ATDD/TDD). Isso inverte a autoridade: a partir daqui, se um teste falha, o bug é da implementação — não do teste.

O que ele faz:

1. Extrai da SDD cada comportamento observável e edge case escrito.
2. Mapeia **critério de aceite → nome do teste**, para que a falha aponte de volta ao critério.
3. Espelha as pastas, factories e o estilo do módulo vizinho — nunca cria framework de teste paralelo.
4. Roda os testes. **Vermelho é o resultado esperado** neste estágio.
5. Entrega um **inventário de testes**: tabela critério → teste → arquivo → comando → status.

Fronteira importante: UI Inertia/Blade/Livewire servida pelo Laravel é testada no ecossistema Laravel, não como SPA.

É **proibido** para ele: inventar comportamento fora da SDD, escrever assert vazio, enfraquecer assertion para acomodar código incompleto, ou implementar a feature.

Sem inventário acionável, os implementadores **não** são liberados.

### 2. `backend-laravel` e/ou `frontend` — implementação (em paralelo quando dá)

Ambos recebem o inventário de testes e implementam até ficar verde. Nenhum dos dois pode tocar nos testes da feature.

**`backend-laravel`** detecta primeiro o modo de entrega — API (rotas `api.php`, Resources, JSON), monolito server-driven (Blade/Livewire/Inertia), ou híbrido — e segue o que o projeto já faz. Implementa **de dentro para fora**: Domain → Application/Service → Infrastructure → Interface. Controllers e componentes Livewire ficam finos; regra de negócio nunca mora no controller nem no Eloquent model. Antes de criar arquivo, espelha um vizinho do mesmo módulo (Policy, grupo de rota, Form Request, naming).

**`frontend`** detecta a stack pelo projeto aberto: `pubspec.yaml` → Flutter; `package.json` com `vue`/`nuxt` → Vue; com `react`/`next` → React. Componentes/widgets ficam finos; orquestração vai para hooks/composables/stores/services no estilo que o módulo já usa. Em caso de ambiguidade (monorepo, React+Vue no mesmo package), escala em vez de chutar.

**A fronteira entre os dois:** UI servida pelo Laravel → `backend-laravel`. App React/Vue/Flutter separado → `frontend`.

Ambos: máximo **3 tentativas** na mesma falha, depois escalam. Ao final, rodam Pint/PHPStan/lint/typecheck se o `AGENTS.md` documentar, e `graphify update .` se o grafo existir.

### 3. `verifier` — validação cética da feature

Roda **somente** os comandos da feature (do inventário ou do `AGENTS.md`) e faz QA contra os aceites. Não roda a suíte do sistema.

Para cada critério da SDD, registra: como validou, resultado (pass / fail / **bloqueado**) e a evidência. Aceite de UI sem ferramenta de browser disponível vira `bloqueado` — nunca `pass` por inferência.

Suspeita de flake? Reexecuta até 2 vezes (3 execuções no total). Se for inconsistente, roteia para o `test-writer` com as três evidências.

Em caso de falha, ele **nomeia o dono** e emite um relatório estruturado:

```
## Falha para <agente>
- Critério da SDD / Comando / Esperado / Atual / Evidência / Causa suspeita / Próximo passo
```

Roteamento: erro de servidor, contrato de API ou falha em Livewire/Inertia → `backend-laravel`. Falha de componente/widget em SPA → `frontend`. Assert desalinhado da SDD ou flake → `test-writer`.

O orquestrador despacha o dono e **re-invoca o verifier**. Só verde libera o próximo estágio.

### 4. `regression` — saúde do sistema inteiro

Só roda **depois** do verifier verde — se não houver confirmação no handoff, ele recusa. Executa a suíte completa (ou o script oficial de regressão do `AGENTS.md`), sem estreitar para "só o que mudou" e sem inventar gates extras. Não reabre o QA da feature.

Mesmas regras de flake e mesmo formato de relatório de falha do verifier.

### 5. `code-reviewer` — review final (readonly)

Só roda com verifier **e** regression confirmados verdes. Não reexecuta suítes: isso já é pré-condição. Foca no **diff**.

O orquestrador é obrigado a passar um pacote de handoff completo: path da SDD, confirmação dos dois gates verdes, base branch (ou comando de diff), stack em escopo, e paths relevantes. Sem isso, o reviewer recusa.

O que ele checa no que o diff tocou: edge fino e regra na camada certa; modo correto (API vs monolito); Form Request + Policy no ponto certo; N+1, transactions, migration nova; contratos estáveis; padrões do frontend sem churn arquitetural; testes intencionais (não vacuamente verdes); e segurança.

Severidades: **Blocker** (segurança, perda de dados, comportamento quebrado vs SDD, violação grave de boundary) → rejeita e roteia ao dono. **Should fix** → recomendação forte. **Nit** → nunca vira Blocker.

### 6. `shipper` — commit, push e PR

Só roda com o code-reviewer **APROVADO**. Faz:

1. Commit na branch da tarefa com a identidade git **já configurada do usuário**.
2. **Sem** trailer `Co-authored-by` e sem co-autoria Cursor/AI.
3. `git push -u origin HEAD`.
4. `gh pr create` com **base** = `development` (ou a base escolhida no estágio 0) e **título** no padrão da branch → PR, ex.: branch `T001-scaffold-laravel` → `[T-001] Scaffold Laravel`.
5. Corpo do PR em português: `## RESUMO` e `## PLANO DE TESTE` (nunca Summary / Test plan).

**Não** devolve “commite e abra o PR manualmente” — a fábrica publica. Devolve a URL do PR no relatório final.

---

## Regras que governam a fábrica

### Hierarquia da verdade

1. A **SDD/spec** define o comportamento pretendido.
2. Os **testes do `test-writer`** são a tradução executável dela.
3. A implementação existe para fazer esses testes passarem.

Conflito entre teste, SDD e código nunca é resolvido enfraquecendo o teste. Vai para o orquestrador com evidência.

### Graphify obrigatório

Todo agente que explora código usa o grafo do projeto **antes** de Read/Grep/Glob:

```bash
graphify query "<pergunta>"      # subgrafo da feature/área
graphify path "<A>" "<B>"        # caminho de dependência entre símbolos
graphify explain "<conceito>"    # nós vizinhos de um conceito
graphify affected "<símbolo>"    # impacto de uma mudança
graphify update .                # após editar produção (AST-only, sem custo de LLM)
```

Isso economiza tokens e revela dependências cross-file que o grep não encontra. Quem edita produção roda `graphify update .` no fim; o `code-reviewer`, sendo readonly, nunca roda `update`.

### Espelhar, nunca inventar

Nenhum agente introduz padrão, pasta, lib de estado, design system ou camada arquitetural que o módulo vizinho não use. A fábrica se adapta ao projeto, não o contrário.

### Nada de contrato inventado

Campos de request/response, props Inertia e campos de formulário só existem se estiverem na SDD ou nos testes. Mudança de contrato necessária mas fora da spec vira escalação, porque afeta o outro lado da entrega.

### Você nunca é o testador

Nenhum agente pede "veja se funcionou". Os especialistas validam com evidência executável.

---

## Pré-requisitos do projeto

Para a fábrica rodar sem travar no estágio 0:

- [ ] **SDD/spec** com critérios de aceite observáveis (não "deve ser rápido", mas "retorna 422 com a mensagem X").
- [ ] **`AGENTS.md`** documentando: comando de teste do backend, do frontend, e da regressão completa. Se houver monorepo, os package paths.
- [ ] **`graphify-out/graph.json`** gerado (opcional, mas é o que torna a exploração barata).
- [ ] Ferramentas de qualidade documentadas se existirem (Pint, PHPStan/Larastan, lint, typecheck).
- [ ] Env/credenciais que a suíte precisa — a fábrica **escala** em vez de marcar verde sem rodar.
- [ ] **`gh` autenticado** no remoto (para o `shipper` abrir o PR).
- [ ] Branch **`development`** no repo (senão o orquestrador pergunta a base uma vez).

---

## Ciclos de correção e escalação

| Situação | Limite |
|----------|--------|
| Implementador na mesma falha | 3 tentativas, depois escala |
| Ciclo verifier → dono → verifier | 3 ciclos, depois escala ao usuário com evidência |
| Ciclo regression → dono → regression | 3 ciclos, depois escala |
| Ciclo code-reviewer BLOQUEADO → dono → re-review | 3 ciclos, depois escala com os Blockers em aberto |
| Reexecução de teste suspeito de flake | 3 execuções, depois roteia ao `test-writer` |

A fábrica só interrompe e chama você por: **ambiguidade de produto**, **env/credenciais faltando**, **branch base indefinida** (sem `development`), **falha do shipper** (auth/`gh`/push), ou **estouro do limite de ciclos**.

---

## Uso avulso dos agentes

Cada agente também funciona isolado, sem o pipeline completo:

- `/test-writer` — atualizar testes quando os critérios de aceite mudarem
- `/backend-laravel` — corrigir um bug de domínio ou endpoint específico
- `/frontend` — ajustar uma tela ou componente
- `/verifier` — validar o estado atual de uma feature contra a spec
- `/regression` — checar a saúde do sistema antes de um release
- `/code-reviewer` — review de um diff (exige gates verdes; recusa review parcial)
- `/shipper` — commit/push/PR após review APROVADO (exige handoff completo)

Para edições de um arquivo só, não use o `orchestrator` — o overhead não compensa.

---

## Limitações conhecidas

- **A fábrica não escreve a SDD.** Ela consome uma spec pronta. Sem critério de aceite testável, ela para no estágio 0.
- **Depende de `AGENTS.md`.** Se o projeto não documenta como rodar os testes, todos os gates escalam em vez de adivinhar comandos.
- **O `shipper` publica no remoto** só após code-reviewer APROVADO; commit sem co-autoria Cursor.
- **Backend é Laravel-only.** Outros backends (Node, Python, Go) não têm agente implementador dedicado.
- **QA de UI depende de ferramenta de browser/app.** Sem ela, aceites UI-facing ficam `bloqueado`, nunca `pass`.
