# Plano Mestre de Orquestração e Execução Autônoma — Desafio Fase 03: Upload e Processamento de Vídeos (StreamTube)

> **Modo de Operação:** Orquestrador de Execução Autônoma com Retomada Determinística (Google Antigravity)  
> **Escopo Exclusivo de Escrita:** Somente arquivos dentro de `./mba-ia-greenfield-project/` (**PROIBIDO** criar ou alterar qualquer arquivo fora da raiz deste repositório)  
> **Branches Autorizadas (Confinamento Estrito):** Apenas `feature/implementa-desafio-dev` (branch atual e base de trabalho) e sua derivada direta `feature/phase-03-videos` (**PROIBIDO** usar ou commitar em qualquer outra branch)  
> **Arquivo de Estado e Retomada:** `docs/desafio/execucao-status.md`  
> **Escopo Funcional:** Fase 03 completa — Backend NestJS 11 + Object Storage S3/MinIO + Fila Redis/BullMQ + Video Worker FFmpeg + Artefatos Auditáveis do Pipeline de IA + Testes Unit/Integration/E2E + Documentação de IA (`next-frontend/` fora de escopo nesta fase)  
> **Referência Normativa:** `docs/desafio/desafio.md`, `.agents/skills/*/SKILL.md`, `.agents/rules/*.md`, `AGENTS.md`, `CLAUDE.md` e `docs/project-plan.md`

---

## Sumário

1. [Visão Geral e Árvore de Entregáveis Auditáveis](#1-visão-geral-e-árvore-de-entregáveis-auditáveis)
2. [Regras Supremas de Governança Autônoma, Precedência, Confinamento e Retomada](#2-regras-supremas-de-governança-autônoma-precedência-confinamento-e-retomada)
   - [2.1. Regra Suprema de Precedência (Skills Designadas > Plano Orquestrador)](#21-regra-suprema-de-precedência-skills-designadas--plano-orquestrador)
   - [2.2. Confinamento Absoluto à Árvore do Projeto (`./mba-ia-greenfield-project/`)](#22-confinamento-absoluto-à-árvore-do-projeto-mba-ia-greenfield-project)
   - [2.3. Confinamento Absoluto de Branches Git](#23-confinamento-absoluto-de-branches-git)
   - [2.4. Mecanismo de Controle de Execução e Retomada (`docs/desafio/execucao-status.md`)](#24-mecanismo-de-controle-de-execução-e-retomada-docsdesafioexecucao-statusmd)
   - [2.5. Rastreabilidade Ponta a Ponta e Evidências de Auditoria de IA](#25-rastreabilidade-ponta-a-ponta-e-evidências-de-auditoria-de-ia)
3. [Operação Nativa no Google Antigravity (Skills, Subagents, MCP e Slug)](#3-operação-nativa-no-google-antigravity-skills-subagents-mcp-e-slug)
4. [Critérios Obrigatórios para Executar e Testar a Aplicação](#4-critérios-obrigatórios-para-executar-e-testar-a-aplicação)
5. [Diretrizes Arquiteturais Pré-Investigadas da Fase 03 (Insumo para as Skills)](#5-diretrizes-arquiteturais-pré-investigadas-da-fase-03-insumo-para-as-skills)
6. [Governança Git Flow e Mapa Granular de Commits](#6-governança-git-flow-e-mapa-granular-de-commits)
7. [Roteiro Passo a Passo de Execução Completa (Etapas `E0` a `E8`)](#7-roteiro-passo-a-passo-de-execução-completa-etapas-e0-a-e8)
   - [Passo Zero (Sempre Executar ao Iniciar ou Retomar) — Protocolo de Retomada](#passo-zero-sempre-executar-ao-iniciar-ou-retomar--protocolo-de-retomada)
   - [Etapa E0 — Pré-flight, Estado Inicial, Criação de `feature/phase-03-videos` e Baseline Verde](#etapa-e0--pré-flight-estado-inicial-criação-de-featurephase-03-videos-e-baseline-verde)
   - [Etapa E1 — Research: Decisões Técnicas da Fase 03 (`skill: research`)](#etapa-e1--research-decisões-técnicas-da-fase-03-skill-research)
   - [Etapa E2 — Planejamento Parte 1: Contexto e 1ª Validação (`plan-context` + `plan-validate`)](#etapa-e2--planejamento-parte-1-contexto-e-1ª-validação-plan-context--plan-validate)
   - [Etapa E3 — Planejamento Parte 2: Resolução, `library-refs.md` via Context7 e Validação `clean` (`plan-resolve` + `plan-validate`)](#etapa-e3--planejamento-parte-2-resolução-library-refsmd-via-context7-e-validação-clean-plan-resolve--plan-validate)
   - [Etapa E4 — Planejamento Parte 3: Plano Executável e Test Specs (`plan-build` + `plan-test-specs`)](#etapa-e4--planejamento-parte-3-plano-executável-e-test-specs-plan-build--plan-test-specs)
   - [Etapa E5 — Implementação Passo a Passo Guiada pelo Plano (`skill: implement`)](#etapa-e5--implementação-passo-a-passo-guiada-pelo-plano-skill-implement)
   - [Etapa E6 — Verificação Empírica Ponta a Ponta em Runtime Real (Docker Compose)](#etapa-e6--verificação-empírica-ponta-a-ponta-em-runtime-real-docker-compose)
   - [Etapa E7 — Atualização e Paridade da Documentação de IA](#etapa-e7--atualização-e-paridade-da-documentação-de-ia)
   - [Etapa E8 — Auditoria Final de Definition of Done, Anti-Reprova e Reintegração na Branch Base](#etapa-e8--auditoria-final-de-definition-of-done-anti-reprova-e-reintegração-na-branch-base)
8. [Checklist Final de Aceite, Rastreabilidade e Anti-Reprova Automática](#8-checklist-final-de-aceite-rastreabilidade-e-anti-reprova-automática)

---

## 1. Visão Geral e Árvore de Entregáveis Auditáveis

Este documento atua como o **orquestrador mestre** para que o agente **Google Antigravity** conduza de forma autônoma, auditável e retomável a implementação completa da **Fase 03 — Upload e Processamento de Vídeos** do **StreamTube**, cumprindo 100% dos requisitos e critérios de avaliação de `docs/desafio/desafio.md`.

Todos os arquivos criados ou alterados residem **exclusivamente** dentro de `./mba-ia-greenfield-project/`:

```text
mba-ia-greenfield-project/
├── AGENTS.md                                                    ← atualizado com seção de vídeos e arquitetura real da Fase 03
├── CLAUDE.md                                                    ← atualizado em paridade estrita com AGENTS.md
├── .agents/
│   └── rules/
│       └── nestjs-project.md                                    ← atualizado com novos serviços Compose, readiness e worker
├── docs/
│   ├── desafio/
│   │   ├── desafio.md                                           ← enunciado normativo oficial (somente leitura)
│   │   ├── relatorio-adaptacao-antigravity.md                   ← relatório de migração Antigravity (somente leitura)
│   │   ├── plano-implementacao-desafio.md                       ← este plano orquestrador (somente leitura na execução)
│   │   └── execucao-status.md                                   ← checkpoint vivo de controle e retomada autônoma
│   ├── diagrams/
│   │   └── software-arch.mermaid                                ← atualizado (Message Queue: Redis / BullMQ)
│   ├── decisions/
│   │   └── technical-decisions-phase-03-videos.md               ← gerado pela skill `research` e decidido via `plan-resolve`
│   └── phases/
│       └── phase-03-videos/                                     ← pasta canônica e obrigatória da Fase 03
│           ├── context.md                                       ← gerado pela skill `plan-context`
│           ├── validation.md                                    ← gerado pela skill `plan-validate` (fechado em `status: clean` + histórico em `Resolved Issues`)
│           ├── library-refs.md                                  ← gerado pela skill `plan-resolve` via Context7 MCP
│           ├── phase-03-videos.md                               ← gerado pela skill `plan-build` (SIs + Tech Specs + Dep Map + Deliverables)
│           └── progress.md                                      ← mantido pela skill `implement` (`Status: completed`)
└── nestjs-project/
    ├── AGENTS.md                                                ← criado/sincronizado em paridade com nestjs-project/CLAUDE.md
    ├── CLAUDE.md                                                ← atualizado (fonte lida por `plan-build` e `implement`)
    ├── Dockerfile.dev                                           ← atualizado com pacote `ffmpeg` (`ffmpeg` + `ffprobe`)
    ├── compose.yaml                                             ← `nestjs-api`, `db`, `mailpit` + `minio`, `redis`, `video-worker`
    ├── .env.example                                             ← atualizado com variáveis de Storage, Fila e Upload
    ├── package.json                                             ← scripts do worker + dependências da Fase 03
    ├── openapi.json                                             ← regenerado via `npm run openapi:export`
    ├── specs/                                                   ← planos `.plan.md` gerados por `plan-test-specs` (quando SIs tiverem `**Test Specs:**`)
    ├── src/
    │   ├── config/                                              ← novos namespaces `registerAs` + schema Joi atualizado
    │   ├── storage/                                             ← módulo S3/MinIO
    │   ├── videos/                                              ← módulo de domínio de vídeos + processamento FFmpeg/BullMQ
    │   ├── worker/ + worker.main.ts                             ← contexto standalone do container `video-worker`
    │   └── database/migrations/<timestamp>-CreateVideos.ts      ← migration versionada gerada via TypeORM CLI
    └── test/
        └── *.e2e-spec.ts                                        ← suíte(s) E2E contra PostgreSQL + MinIO + Redis + Worker reais
```

---

## 2. Regras Supremas de Governança Autônoma, Precedência, Confinamento e Retomada

As cinco regras abaixo são **invariantes absolutos**. Qualquer violação a qualquer uma delas invalida a execução.

### 2.1. Regra Suprema de Precedência (Skills Designadas > Plano Orquestrador)

Em caso de qualquer ambiguidade, divergência de formato, template de seção, nome de campo em frontmatter, critério de elegibilidade de TD, regra de decomposição/auto-split de SIs ou convenção de testes entre o que foi inferido neste plano orquestrador, o enunciado do desafio e as skills do repositório, aplica-se **obrigatoriamente** esta ordem de precedência (do nível mais alto ao mais baixo):

1. **Nível 1 (Inviolável — Segurança, Escopo e Aceite):**
   - Confinamento absoluto de escrita à árvore `./mba-ia-greenfield-project/` (§ 2.2) e confinamento absoluto de branches a `feature/implementa-desafio-dev` e `feature/phase-03-videos` (§ 2.3).
   - Requisitos obrigatórios, Critérios de Aceite, estrutura da pasta da fase (`docs/decisions/technical-decisions-phase-03-videos.md` e `docs/phases/phase-03-videos/phase-03-videos.md`) e cláusulas de Reprova Automática de `docs/desafio/desafio.md`.
   - Regras globais e modulares em `AGENTS.md` e `.agents/rules/*.md`.
2. **Nível 2 (Soberano sobre Processo e Formato — Skills Designadas do Repositório):**
   - **TODOS os padrões, contratos de entrada/saída, filtros, gates, templates e fluxos definidos nas skills em `.agents/skills/*/SKILL.md`** (`research`, `plan-pipeline`, `plan-context`, `plan-validate`, `plan-resolve`, `plan-build` + `phase-a.md` + `phase-b.md` + `templates/tech-specs/*.md`, `plan-test-specs`, `implement`, `testing-guide-nestjs-project`, `nestjs-best-practices`, `typeorm`) e nos subagents em `.agents/agents/*.md` **PREVALECEM INTEGRALMENTE** sobre qualquer formato, rota, coluna, numeração ou lista de SIs sugerida como referência neste `plano-implementacao-desafio.md`!
   - *Exemplos práticos desta precedência:*
     - Se o filtro `(d)+(a)` de `research/SKILL.md` ou os checks `MD-N`/`IC-N`/`AMB-N` de `plan-validate/SKILL.md` exigirem fundir, desmembrar ou ajustar qualquer TD sugerido na Seção 5, **siga `research` e `plan-validate`**.
     - Se `plan-build` (`phase-a.md` / `phase-b.md`) exigir dividir SIs de controller wiring com `**Route:**` + `**Test Specs:** _pending /plan-test-specs_`, suprimir linhas E2E inline nesses SIs (delegando-as para `plan-test-specs`) ou aplicar auto-split (`SI-03.X.1` / `SI-03.X.2`) pelo teto de `≤ 5 Technical actions`, `≤ 5 Test files` e `≤ 10 Acceptance criteria`, **siga estritamente `plan-build` e `plan-test-specs`**.
3. **Nível 3 (Artefatos Gerados pelo Pipeline):**
   - Durante a Etapa E5 (`implement`), **o documento `docs/phases/phase-03-videos/phase-03-videos.md` gerado por `plan-build` é o único contrato de SIs**. Implemente exatamente os SIs, ações técnicas, specs e testes definidos em `phase-03-videos.md`.
4. **Nível 4 (Referência Arquitetural deste Orquestrador):**
   - As decisões e os SIs descritos nas Seções 5 e 7 deste arquivo fornecem a engenharia de domínio pré-investigada no código atual (prevenindo armadilhas reais do projeto), atuando como guia de conteúdo subordinado aos Níveis 1, 2 e 3.

---

### 2.2. Confinamento Absoluto à Árvore do Projeto (`./mba-ia-greenfield-project/`)

**JAMAIS CRIE, EDITE, MOVA OU APAGUE QUALQUER ARQUIVO FORA DA RAIZ DESTE PROJETO:**
- Raiz Windows permitida: `\\wsl.localhost\Ubuntu\home\c\FullCycle\desafios\mba-ia-greenfield-project\`
- Raiz Linux/WSL permitida: `/home/c/FullCycle/desafios/mba-ia-greenfield-project/`

Regras operacionais decorrentes:
1. **Proibido escrever em `<appDataDir>\brain\...` (`C:\Users\c\.gemini\antigravity\brain\...`):** Não crie artefatos `.md` de conversa nem scripts em `scratch/`. Todo controle de progresso e tarefas vive dentro do projeto em `docs/desafio/execucao-status.md` e `docs/phases/phase-03-videos/progress.md`.
2. **Proibido tocar em `~/.gemini/config/...` ou diretórios de usuário/sistema:** Nunca edite configurações globais da máquina. Leitura somente-leitura de schemas MCP (como `context7_context7`) e skills built-in é permitida; escrita fora da raiz é **terminantemente proibida**.
3. **Proibido gravar arquivos temporários em `/tmp` ou `os.tmpdir()`:** Nos testes de integração e E2E que precisam de um vídeo `.mp4` real ou corrompido, gere o `.mp4` **diretamente em memória (`Buffer`)** via `ffmpeg ... -movflags frag_keyframe+empty_moov -f mp4 pipe:1` (`encoding: 'buffer'`). Caso algum teste precise estritamente de arquivo físico temporário, grave apenas em `nestjs-project/.tmp-test/` (adicionado ao `nestjs-project/.gitignore` e removido no `afterAll`).
4. **Volumes Docker:** No `nestjs-project/compose.yaml`, utilize apenas bind-mount interno ao projeto (`.:/home/node/app`) e volumes nomeados gerenciados pelo Docker (`minio_data:/data`), nunca caminhos de host externos ao repositório.

---

### 2.3. Confinamento Absoluto de Branches Git

1. **Únicas duas branches permitidas em toda a execução:**
   - **`feature/implementa-desafio-dev`** — branch atual e **branch base de trabalho** (já derivada de `dev` e atualizada com merge de `main`). Recebe apenas o commit inicial de setup do orquestrador (`C0`, caso haja mudanças pendentes em `docs/desafio/`) e o merge final `--no-ff` de encerramento (`C16`).
   - **`feature/phase-03-videos`** — derivada **diretamente** de `feature/implementa-desafio-dev` (`git checkout -b feature/phase-03-videos`). Recebe **todos** os commits de planejamento, implementação e documentação da Fase 03 (`C1` até `C15+`).
2. **Proibição Total sobre Demais Branches:**
   - É **TERMINANTEMENTE PROIBIDO** executar `git checkout`, `git switch`, `git commit`, `git merge`, `git rebase`, `git cherry-pick`, `git reset` ou `git branch -f/-D` em `main`, `dev` ou qualquer outra branch existente no repositório.
   - Em chamadas `invoke_subagent`, use **sempre** `Workspace: "inherit"` (jamais `"branch"` ou `"share"`).
3. **Trava Mecânica Pré-Commit (obrigatória antes de TODO `git commit` ou `git merge`):**
   - Antes de cada commit, execute `git branch --show-current` e confirme que a saída é literalmente `feature/phase-03-videos` (em `C1..C15+`) ou `feature/implementa-desafio-dev` (apenas em `C0` e `C16`). Se for qualquer outro valor, **PARE IMEDIATAMENTE** e reporte erro ao usuário sem commitar.
   - Na Etapa `E0`, registre os hashes atuais de `main` e `dev` (`git rev-parse main dev`) em `docs/desafio/execucao-status.md`; na Etapa `E8`, comprove via `git rev-parse main dev` que ambos permanecem **intocados**.

---

### 2.4. Mecanismo de Controle de Execução e Retomada (`docs/desafio/execucao-status.md`)

Para garantir que a execução autônoma possa ser interrompida a qualquer instante (limite de contexto, falha de rede, reinício de sessão, pausa manual) e retomada sem perda, duplicação ou corrupção de trabalho, o agente manterá o arquivo **`docs/desafio/execucao-status.md`**.

#### Template Canônico de `docs/desafio/execucao-status.md`

```markdown
---
plan_file: docs/desafio/plano-implementacao-desafio.md
base_branch: feature/implementa-desafio-dev
work_branch: feature/phase-03-videos
immutable_refs:
  main_sha: "<capturado em E0 via git rev-parse main>"
  dev_sha: "<capturado em E0 via git rev-parse dev>"
overall_status: in_progress
current_stage: E0
current_substep: E0.1
last_completed_step: none
last_commit_sha: "<capturado após cada commit>"
updated_at: "<ISO-8601>"
---

# Controle de Execução Autônoma — Fase 03 (StreamTube)

## Progresso das Macro-Etapas

| Etapa | Descrição | Status | Artefato Chave / Evidência | Commit SHA |
| :---: | :--- | :---: | :--- | :---: |
| **E0** | Pré-flight, estado inicial, branch `feature/phase-03-videos` e baseline verde | `pending` | Branch criada + suíte Fases 01–02 verde | — |
| **E1** | Research (`skill: research`) | `pending` | `docs/decisions/technical-decisions-phase-03-videos.md` (`pending`) | — |
| **E2** | Contexto + 1ª Validação (`plan-context` + `plan-validate`) | `pending` | `context.md` + `validation.md` (`dirty` com `OQ-N`) | — |
| **E3** | Resolução + Context7 + Validação `clean` (`plan-resolve` + `plan-validate`) | `pending` | `library-refs.md` + `validation.md` (`status: clean`) | — |
| **E4** | Plano Executável + Test Specs (`plan-build` + `plan-test-specs`) | `pending` | `phase-03-videos.md` (+ `nestjs-project/specs/*.plan.md` se aplicável) | — |
| **E5** | Implementação SI a SI (`skill: implement`) | `pending` | `docs/phases/phase-03-videos/progress.md` (`Status: completed`) | — |
| **E6** | Verificação Empírica Ponta a Ponta em Runtime Real | `pending` | Evidência de upload multipart + worker + HTTP `206` + download | — |
| **E7** | Atualização e Paridade da Documentação de IA | `pending` | `AGENTS.md`, `CLAUDE.md`, `nestjs-project/*`, `software-arch.mermaid` | — |
| **E8** | Auditoria Final DoD, Anti-Reprova e Merge `--no-ff` em `feature/implementa-desafio-dev` | `pending` | DoD 100% verde + merge concluído + `main`/`dev` intocadas | — |

## Progresso Granular de SIs na Etapa E5 (Espelho de `docs/phases/phase-03-videos/progress.md`)

_(Populado ao iniciar E5 com a lista exata de SIs gerada em `docs/phases/phase-03-videos/phase-03-videos.md`; atualizado a cada SI concluído com o respectivo Commit SHA.)_

## Diário de Checkpoints e Retomadas

- `<ISO-8601>` — `[E0]` Inicialização do controle de execução.
```

#### Regra de Atomicidade de Checkpoint
- Atualize `docs/desafio/execucao-status.md` **imediatamente antes** de cada commit Git e inclua `docs/desafio/execucao-status.md` no `git add` do próprio commit! Assim, cada commit Git carrega em si o ponteiro exato da etapa concluída (zero arquivo dirty órfão entre etapas; em `last_commit_sha`, registre o SHA do commit imediatamente anterior e a etapa que este commit está fechando).

---

### 2.5. Rastreabilidade Ponta a Ponta e Evidências de Auditoria de IA

O avaliador verificará que a entrega foi genuinamente produzida pelo workflow de IA do repositório (`docs/desafio/desafio.md`, linhas 29–43 e 159–195). Garanta **obrigatoriamente** as seguintes evidências orgânicas no repositório e no histórico Git:

1. **Rastreabilidade `project-plan.md` → `technical-decisions-phase-03-videos.md`:** Todo TD cita literalmente em `**Capability:**` um ou mais dos 9 bullets da Fase 03 em `docs/project-plan.md`. Em `E1`, o arquivo é commitado (`C1`) com `status: pending` e `**Decision:** _[pending]_` (como manda `research/SKILL.md`).
2. **Rastreabilidade `context.md` → `validation.md` (`dirty` → `clean`):**
   - Em `E2` (`C2`), `validation.md` registra `status: dirty` listando as questões abertas (`OQ-N` para os TDs pendentes e quaisquer outras levantadas).
   - Em `E3` (`C3`), `plan-resolve` move cada questão resolvida para `## Resolved Issues` com `resolved_by: phase-03-videos/TD-XX`, preenche `Decision:` + `Libraries:` no documento de decisões, grava `library-refs.md` com `context7_id` e trechos reais da doc oficial obtidos via MCP Context7, e a nova rodada de `plan-validate` grava `status: clean`, `issue_count: 0` **preservando intacta a seção `## Resolved Issues`** (prova documental auditável do ciclo `validate ↔ resolve`).
3. **Rastreabilidade `context.md` + `library-refs.md` → `phase-03-videos.md` (+ `specs/*.plan.md`):**
   - Em `E4` (`C4`), cada ação técnica nos SIs referencia inline o TD correspondente (`(per phase-03-videos/TD-XX)`), e `## Technical Specifications` contém obrigatoriamente `### Data Model`, `### API Contracts`, `### Authorization Matrix`, `### Error Catalog` e **`### Events/Messages`** (exigido pelo desafio por causa da fila).
4. **Rastreabilidade `phase-03-videos.md` → Código + Testes + `progress.md`:**
   - Em `E5`, cada SI concluído atualiza `docs/phases/phase-03-videos/progress.md` com a contagem exata de testes executados e observações reais encontradas, gerando 1 commit dedicado no Git por SI.
5. **Rastreabilidade Código Real → `AGENTS.md` / `CLAUDE.md`:**
   - Em `E7`, nenhum arquivo, rota, script ou container citado na documentação pode divergir do que existe fisicamente no disco.

---

## 3. Operação Nativa no Google Antigravity (Skills, Subagents, MCP e Slug)

### 3.1. Tabela de Tradução de Ferramentas (Skills → Antigravity)

| Primitiva citada na Skill | Equivalente Exato no Antigravity | Regra de Uso em Modo Autônomo (e Confinamento) |
| :--- | :--- | :--- |
| `Skill` tool / `/research`, `/plan-*`, `/implement` | `read_file` em `.agents/skills/<skill>/SKILL.md` e seus arquivos companheiros (`phase-a.md`, `phase-b.md`, `references/*`, `templates/*`, `artifacts/*`) | **Obrigatório** ler o `SKILL.md` correspondente antes de iniciar cada etapa. Nunca execute uma etapa "de memória". |
| `mcp__context7__resolve-library-id` e `mcp__context7__query-docs` | `call_mcp_tool` com `ServerName: "context7_context7"` e `ToolName: "resolve-library-id"` / `"query-docs"` | Leia primeiro (somente leitura) os schemas JSON do servidor MCP `context7_context7` e a skill `context7-mcp`, e execute as consultas reais para cada biblioteca nova (`@nestjs/bullmq`, `bullmq`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`) e qualquer dúvida de API NestJS 11 / TypeORM 0.3. |
| `Agent` com leitores em `.agents/agents/*.md` (`plan-reader`, `decisions-reader`, `decisions-detail-reader`, `decisions-correlator`, `phases-reader`) | `define_subagent` (`EnableWriteTools: false`) + `invoke_subagent` (`Workspace: "inherit"`) | Leia `.agents/agents/<nome>.md`, registre via `define_subagent` na conversa atual e dispare em paralelo via `invoke_subagent` (`Workspace: "inherit"`). Encerre o turno sem polling para receber os retornos via wakeup reativo. |
| `Agent` com `Explore` | `invoke_subagent` com `TypeName: "research"`, `Workspace: "inherit"` | Solicite resumos enxutos/assinaturas conforme contrato de `implement/SKILL.md`. |
| `Agent` com `general-purpose` (execução de testes em `implement` Step 4) | `invoke_subagent` com `TypeName: "self"`, `Workspace: "inherit"` (ou execução direta via `run_command` quando conciso) | Se delegar ao `self`, proíba explicitamente qualquer escrita de arquivo ou comando Git no subagent — ele deve apenas ler `nestjs-project/CLAUDE.md`, rodar os testes solicitados via WSL Docker (`BypassSandbox: true`, `-T`) e retornar o relatório diagnóstico filtrado. |
| `AskUserQuestion` | Em modo autônomo autorizado: resolução automática pelas escolhas pré-aprovadas na Seção 5; em bloqueio genuíno imprevisto: `ask_question` | Em modo autônomo: (a) em `plan-context` Step 4, confirme correlação `high` (`openapi-docs-nestjs`); (b) em `plan-resolve` Step 3, aplique as opções recomendadas da Seção 5; (c) em `plan-build` `A5`, selecione `"Continue now (Phase B in this session)"`; (d) em `implement`, opere em modo `continuous` (nativo de `implement/SKILL.md`). Só acione `ask_question` se surgir conflito arquitetural inédito impossível de deduzir com segurança. |
| `TaskCreate` / `TaskUpdate` | Seção `## Progresso Granular de SIs na Etapa E5` em `docs/desafio/execucao-status.md` + `docs/phases/phase-03-videos/progress.md` | **NUNCA** crie arquivos em `<appDataDir>\brain\...`. Mantenha o rastreamento 100% dentro de `docs/desafio/execucao-status.md` e `docs/phases/phase-03-videos/progress.md`. |

### 3.2. Regra Crítica de Nomenclatura de Slug (`phase-03-videos`)

`docs/desafio/desafio.md` fixa literalmente no contrato de avaliação:
- Decisões técnicas: `docs/decisions/technical-decisions-phase-03-videos.md`
- Pasta da fase: `docs/phases/phase-03-videos/`
- Plano da fase: `docs/phases/phase-03-videos/phase-03-videos.md`

Esse formato espelha `phase-01-configuracao-base`, `phase-02-auth` e `phase-02-auth-frontend`. Portanto:
1. Em `research`, salve **exatamente** em `docs/decisions/technical-decisions-phase-03-videos.md` (mantendo `phase-03-videos` após `technical-decisions-`).
2. Os identificadores `Ref` emitidos por `decisions-reader` / `decisions-detail-reader` serão `phase-03-videos/TD-01`, `phase-03-videos/TD-02`, etc.
3. Em todos os estágios (`plan-context`, `plan-validate`, `plan-resolve`, `plan-build`, `plan-test-specs`, `implement`), deduplicar o prefixo `phase-03-` ao montar o caminho da pasta: `{target_dir}` é **`docs/phases/phase-03-videos/`** (jamais `docs/phases/phase-03-phase-03-videos/`!), frontmatter `name: phase-03-videos`, e arquivo final `docs/phases/phase-03-videos/phase-03-videos.md`.

---

## 4. Critérios Obrigatórios para Executar e Testar a Aplicação

### 4.1. Execução via WSL2 + Docker Compose (`BypassSandbox: true` e `-T`)

1. O binário `docker` reside na distro WSL `Ubuntu` (`Docker 28.1.1` / `Compose v2.35.1`), não no PowerShell do Windows, e o sandbox padrão do Windows bloqueia `wsl.exe` (`Wsl/E_ACCESSDENIED`).
2. Todo comando Docker/container ou Git no WSL deve usar formato direto e auto-aprovável com `BypassSandbox: true` e flag `-T` em `docker compose exec`:
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api <comando>
   ```
3. **Regra Estrita Container-Only:** Todo `npm`, `npx`, `node`, `tsc`, `typeorm` e `jest` roda **exclusivamente dentro do container `nestjs-api`**, nunca no host.
4. **Docker Networking:** Comunicação entre containers usa sempre o nome do serviço no `compose.yaml` (`DB_HOST=db`, `MAIL_HOST=mailpit`, `REDIS_HOST=redis`, `STORAGE_ENDPOINT=http://minio:9000`), **jamais `localhost` ou `127.0.0.1`**. Valores com `<`, `>` ou espaços em `.env` / `.env.example` devem estar entre aspas duplas.
5. **Servidor HTTP:** `docker compose up -d` sobe `db`, `mailpit`, `minio`, `redis`, `video-worker` e mantém `nestjs-api` em `tail -f /dev/null`. Não deixe `npm run start:dev` rodando em background; na verificação HTTP da Etapa `E6`, suba temporariamente o processo compilado, valide e encerre-o imediatamente.

### 4.2. Critérios de Prontidão (Readiness) dos 6 Serviços no Compose

1. `docker compose ps` → `nestjs-api`, `db`, `mailpit`, `minio`, `redis` e `video-worker` com status `running` (e `healthy` onde configurado).
2. `docker compose exec -T db pg_isready -U streamtube` → `accepting connections`.
3. `docker compose exec -T redis redis-cli ping` → `PONG`.
4. `docker compose exec -T nestjs-api curl -sf http://minio:9000/minio/health/live` → exit `0`.
5. `docker compose exec -T nestjs-api ffmpeg -version` e `docker compose exec -T video-worker ffprobe -version` → exit `0`.
6. `docker compose logs --tail=30 video-worker` → `WorkerModule` inicializado e escutando a fila sem erros.

### 4.3. Pirâmide de Testes, Infraestrutura Real e Prevenção de Regressões

1. **Infraestrutura Real Obrigatória (Override sobre nota legada da Fase 01):** `docs/desafio/desafio.md` (linhas 120, 146, 175, 191 e 243) reprova automaticamente qualquer entrega sem fila, worker e storage S3/MinIO reais no `compose.yaml` ou que simule em filesystem local o que pode rodar nos containers. Testes `*.integration-spec.ts` e `*.e2e-spec.ts` **DEVEM** exercitar **PostgreSQL real (`db`)**, **MinIO real (`http://minio:9000`)**, **Redis/BullMQ real (`redis:6379`)** e **FFmpeg/ffprobe reais**.
2. **Sufixos e Regras do `testing-guide-nestjs-project`:**
   - `*.spec.ts` (em `src/`): testes unitários com mocks de fronteira DI + testes de compilação de módulos (`*.module.spec.ts`). **Nunca** escreva testes unitários para controllers.
   - `*.integration-spec.ts` (em `src/`): contratos reais com DB, MinIO, Redis/BullMQ e FFmpeg/ffprobe.
   - `*.e2e-spec.ts` (em `test/`): ciclo HTTP completo via `supertest` reproduzindo `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`, `DomainExceptionFilter` e `ValidationExceptionFilter`.
3. **Geração de Fixture `.mp4` em Memória (Zero Escrita fora do Projeto):** Gere o `.mp4` de teste diretamente em um `Buffer` em RAM invocando `execFile('ffmpeg', ['-f', 'lavfi', '-i', 'testsrc=duration=1.5:size=320x240:rate=15', '-c:v', 'libx264', '-pix_fmt', 'yuv420p', '-movflags', 'frag_keyframe+empty_moov', '-f', 'mp4', 'pipe:1'], { encoding: 'buffer', maxBuffer: 10 * 1024 * 1024 })`. Nenhum arquivo é criado em `/tmp` ou `os.tmpdir()`.
4. **Atualização Obrigatória dos 11 Arquivos `ALL_ENTITIES` + `cleanAllTables`:**
   - Ao declarar `@OneToMany(() => Video, (video) => video.channel)` em `Channel` (regra de relação bidirecional de `.agents/rules/nestjs-entities.md`), inclua `Video` nos **11 arquivos** existentes que instanciam `createTestDataSource` (`users.module.spec.ts`, `users.service.integration-spec.ts`, `user.entity.integration-spec.ts`, `channels.module.spec.ts`, `channels.service.integration-spec.ts`, `channel.entity.integration-spec.ts`, `auth.module.spec.ts`, `auth.service.integration-spec.ts`, `refresh-token.entity.integration-spec.ts`, `verification-token.entity.integration-spec.ts` e `migrations.integration-spec.ts`) e adicione `await dataSource.query('DELETE FROM "videos"');` **na primeira linha** de `cleanAllTables()` em `src/test/create-test-data-source.ts` (antes de `channels` e `users`).
   - Em `src/database/migrations.integration-spec.ts`, adicione `'videos'` em `MANAGED_TABLES`, importe explicitamente a classe da migration `CreateVideos...`, adicione `DROP TYPE IF EXISTS "public"."videos_status_enum" CASCADE`, atualize `toHaveLength(3)` e garanta `await dataSource.runMigrations()` no `afterAll`.
5. **Tratamento do `ThrottlerGuard` Global (`10 req/min` em `AuthModule`):**
   - Marque rotas de leitura contínua (`stream`, `download`, `thumbnail`, `presign` de partes e consulta `GET /videos/:shortCode`) com `@SkipThrottle()` para não bloquear scrubbing de player HTML5 nem uploads multi-parte.
   - Nos testes `*.e2e-spec.ts`, injete `ThrottlerStorage` e execute `throttlerStorage.storage.clear()` no `beforeEach`.
6. **Isolamento de Fila entre Testes de Integração de Publisher e o Container `video-worker`:**
   - Como `video-worker` consome continuamente a fila `video-processing`, testes de integração que verificam jobs enfileirados antes do consumo devem usar uma fila isolada de teste via configuração do módulo de teste (ou pausar a fila no teste e retomar no `afterAll`), enquanto os testes E2E usam a fila real `video-processing` para validar o processamento ponta a ponta pelo `video-worker`!
7. **Fechamento de Recursos e Execução Serial:** Sempre rodar com `--runInBand` e encerrar `app.close()`, `module.close()`, `dataSource.destroy()` e `s3Client.destroy()` em `afterAll`.

### 4.4. Definition of Done (DoD) Técnica Completa

Todos os comandos abaixo (executados via WSL dentro do container `nestjs-api`) devem sair com código `0`:
1. `docker compose exec -T nestjs-api npm run migration:run`
2. `docker compose exec -T nestjs-api npm test`
3. `docker compose exec -T nestjs-api npm run test:e2e`
4. `docker compose exec -T nestjs-api npx tsc --noEmit`
5. `docker compose exec -T nestjs-api npm run build`
6. `docker compose exec -T nestjs-api npm run lint`
7. `docker compose exec -T nestjs-api npm run openapi:export` *(sem deixar diff não commitado em `nestjs-project/openapi.json`)*

---

## 5. Diretrizes Arquiteturais Pré-Investigadas da Fase 03 (Insumo para as Skills)

> **Nota de Precedência (§ 2.1):** O catálogo abaixo sintetiza a investigação técnica realizada sobre o código existente para alimentar `research`, `plan-resolve` e `plan-build`. As skills têm autonomia e precedência para refinar nomes, agrupar/desmembrar TDs ou ajustar templates conforme seus contratos formais.

### 5.1. Decisões Técnicas de Referência (`TD-01` a `TD-08`)

1. **`TD-01` — Tecnologia de Fila (`Scope: Backend`):**
   - *Capability:* `Serviço de processamento em segundo plano (filas)`
   - *Opções:* **A)** BullMQ + Redis (`@nestjs/bullmq`, `bullmq`, container `redis:7-alpine`) **[Recomendada/Escolhida]** vs **B)** RabbitMQ (`@nestjs/microservices` RMQ + `amqplib`) vs **C)** `pg-boss` sobre PostgreSQL (descartada por acoplar carga ao DB e não prover serviço dedicado de fila no Compose).
   - *Libraries:* `@nestjs/bullmq, bullmq`
2. **`TD-02` — SDK de Object Storage, Topologia de Bucket/Chaves e Assinatura (`Scope: Backend`):**
   - *Capability:* `Serviço de armazenamento de arquivos (vídeos e thumbnails)`
   - *Opções:* **A)** AWS SDK v3 (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, `forcePathStyle: true`) com bucket único (`streamtube`) organizado por prefixos (`videos/{channelId}/{videoId}/...` e `thumbnails/{channelId}/{videoId}/...`), provisionamento idempotente em `OnModuleInit` e separação entre `STORAGE_ENDPOINT` (`http://minio:9000`) e `STORAGE_PUBLIC_ENDPOINT` opcional **[Recomendada/Escolhida]** vs **B)** Múltiplos buckets separados vs **C)** SDK proprietário `minio`.
   - *Libraries:* `@aws-sdk/client-s3, @aws-sdk/s3-request-presigner`
3. **`TD-03` — Estratégia de Upload de até 10GB sem Travar a API + Pré-Cadastro em Rascunho (`Scope: Cross-layer`):**
   - *Capability:* `Transversal — covers: "Upload de vídeos com suporte a arquivos de até 10GB sem impacto na performance", "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload"`
   - *Opções:* **A)** S3 Presigned Multipart Upload orquestrado pela API (`POST /videos/uploads` valida `sizeBytes <= 10 GiB`, cria `Video` em `status = 'draft'` com `short_code` único, abre `CreateMultipartUpload` e retorna `uploadId` + URLs pré-assinadas das partes; cliente faz `PUT` dos chunks direto ao MinIO/S3 sem trafegar vídeo pela API; `POST /videos/:id/complete` finaliza o multipart, muda para `processing` e enfileira job; `DELETE /videos/:id/upload` aborta partes órfãs; rota de re-assinatura de parte suporta retomada) **[Recomendada/Escolhida]** vs **B)** Single Presigned PUT (inviável: limite S3 de 5 GiB e sem retomada) vs **C)** Servidor `tus` intermediário vs **D)** Upload streaming via API NestJS (reprova automática se travar/segurar a API).
4. **`TD-04` — Topologia de Execução do Video Worker (`Scope: Backend`):**
   - *Capability:* `Serviço de processamento em segundo plano (filas)`
   - *Opções:* **A)** Standalone NestJS Application Context (`NestFactory.createApplicationContext(WorkerModule)` em `src/worker.main.ts`) compartilhando entidades/configs do `nestjs-project/` e rodando em container dedicado `video-worker` no `compose.yaml` **[Recomendada/Escolhida]** vs **B)** Worker in-process na API HTTP vs **C)** Microserviço em pacote/repositório separado.
5. **`TD-05` — Extração de Metadados e Geração de Thumbnail para até 10GB (`Scope: Backend`):**
   - *Capability:* `Transversal — covers: "Processamento automático do vídeo após upload (extração de duração e metadados)", "Geração automática de thumbnail a partir de um frame do vídeo"`
   - *Opções:* **A)** Invocação direta de `ffprobe` e `ffmpeg` via `node:child_process` (`execFile` sem shell) lendo Presigned S3 GET URL via HTTP `Range` (sem baixar 10 GB para disco local; seek adaptativo `duration > 1 ? Math.min(Math.floor(duration * 0.1), 5) : 0` gerando JPEG e enviando ao MinIO) **[Recomendada/Escolhida]** vs **B)** Download integral para `/tmp` local vs **C)** `fluent-ffmpeg` (biblioteca arquivada/depreciada em 2025).
6. **`TD-06` — Estratégia de URL Única por Vídeo (`Scope: Cross-layer`):**
   - *Capability:* `URL única por vídeo, sem conflito com outros vídeos`
   - *Opções:* **A)** `short_code` aleatório URL-safe de 11 caracteres (`[A-Za-z0-9_-]`, estilo YouTube) via CSPRNG nativo (`node:crypto`) + constraint `UNIQUE` no PostgreSQL + retry em violação `23505` **[Recomendada/Escolhida]** vs **B)** Slug textual derivado de título vs **C)** UUIDv4 bruto de 36 chars na URL.
7. **`TD-07` — Estratégia de Streaming (`206 Partial Content`) e Download (`Scope: Cross-layer`):**
   - *Capability:* `Transversal — covers: "Reprodução via streaming (sem necessidade de download completo)", "Download do vídeo pelo usuário"`
   - *Opções:* **A)** Streaming HTTP `Range` (`206 Partial Content`, `Accept-Ranges: bytes`, `Content-Range`, `416` em faixa inválida, `200` fallback sem `Range`) e Download (`Content-Disposition: attachment`) via pipe não-bufferizado (`Readable`, memória O(1)) de `GetObjectCommand` do S3 em `GET /videos/:shortCode/stream`, `GET /videos/:shortCode/download` e `GET /videos/:shortCode/thumbnail` **[Recomendada/Escolhida]** vs **B)** `302 Redirect` para Presigned URL (atrito de `Host` SigV4 `minio:9000` vs `localhost:9000` sem reverse proxy) vs **C)** Transcodificação HLS/DASH (fora de escopo da Fase 03).
8. **`TD-08` — Máquina de Estados de Status e Política de Falha no Processamento (`Scope: Backend`):**
   - *Capability:* `Transversal — covers: "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload", "Processamento automático do vídeo após upload (extração de duração e metadados)", "Serviço de processamento em segundo plano (filas)"`
   - *Opções:* **A)** Enum PostgreSQL `draft → processing → ready | failed` na tabela `videos` (com `error_reason`), retentativas BullMQ `attempts: 3` com backoff exponencial (`1000ms`) para falhas transitórias, `UnrecoverableError` imediato para mídia inválida/objeto ausente e `@OnWorkerEvent('failed')` persistindo `status = 'failed'` sem derrubar o worker **[Recomendada/Escolhida]** vs **B)** Falha sem retry (`attempts: 1`) vs **C)** Tabela separada de eventos sem coluna de status direta.

### 5.2. Resumo da Superfície Técnica Alvo (Alinhada aos Templates de `plan-build`)

- **Entidades (`### Data Model`):**
  - `Video` (`videos`): `id` (`uuid` PK), `channel_id` (`uuid` FK → `channels.id` `ON DELETE CASCADE`), `short_code` (`varchar(16)` unique), `title` (`varchar(255)`), `status` (`enum`: `'draft' | 'processing' | 'ready' | 'failed'`, default `'draft'`), `original_filename` (`varchar(255)`), `mime_type` (`varchar(100)`), `size_bytes` (`bigint`), `storage_key` (`varchar(512)`), `upload_id` (`varchar(512)` nullable, `select: false`), `thumbnail_key` (`varchar(512)` nullable), `duration_seconds` (`int` nullable), `width` (`int` nullable), `height` (`int` nullable), `video_codec` (`varchar(64)` nullable), `error_reason` (`text` nullable), `created_at`, `updated_at`.
  - `Channel` (`channels`): relação inversa `@OneToMany(() => Video, (video) => video.channel) videos: Video[]`.
- **Rotas REST (`### API Contracts` & `### Authorization Matrix`):**
  - `POST /videos/uploads` (Authenticated) → `201` (pré-cadastra `draft` + inicia S3 Multipart + retorna URLs das partes; valida até `10 GiB = 10737418240` bytes).
  - `POST /videos/:id/parts/:partNumber/presign` (Owner, `@SkipThrottle()`) → `200` (re-assina parte para retomada).
  - `POST /videos/:id/complete` (Owner) → `200` (completa multipart no S3, transiciona para `processing`, publica job `process-video`).
  - `DELETE /videos/:id/upload` (Owner) → `204` (aborta multipart no S3 e remove rascunho incompleto).
  - `GET /videos/:shortCode` (Anonymous `@Public()`, `@SkipThrottle()`) → `200` (metadados, status e links de mídia).
  - `GET /videos/:shortCode/stream` (Anonymous `@Public()`, `@SkipThrottle()`) → `206 Partial Content` (com `Range`) / `200 OK` (sem `Range`) / `416` (faixa inválida) / `409 VIDEO_NOT_READY`.
  - `GET /videos/:shortCode/download` (Anonymous `@Public()`, `@SkipThrottle()`) → `200 OK` (`Content-Disposition: attachment`).
  - `GET /videos/:shortCode/thumbnail` (Anonymous `@Public()`, `@SkipThrottle()`) → `200 OK` (`image/jpeg`).
- **Fila (`### Events/Messages`):** Fila BullMQ `video-processing`, job `process-video`, payload `{ "videoId": "uuid" }`, `Producer: VideosService`, `Consumer: VideoProcessor`, `Delivery semantics: at-least-once` (`attempts: 3`, backoff exponencial `1000ms`, `jobId: videoId`).

---

## 6. Governança Git Flow e Mapa Granular de Commits

1. **Confinamento Estrito (§ 2.3):** Todos os commits ocorrem **exclusivamente** em `feature/implementa-desafio-dev` (`C0` e `C16`) e em sua derivada direta `feature/phase-03-videos` (`C1` até `C15+`). **Jamais** toque em `main`, `dev` ou qualquer outra branch.
2. **Granularidade:** Cada etapa do pipeline e **cada SI individual concluído com suíte verde** em `implement` gera seu próprio commit semântico, curto, descritivo, focado no "porquê", contendo também a atualização sincronizada de `docs/desafio/execucao-status.md`.

| Código | Branch Permitida | Gatilho de Commit | Mensagem Padrão |
| :---: | :--- | :--- | :--- |
| **C0** | `feature/implementa-desafio-dev` | Inicialização de `execucao-status.md` + `plano-implementacao-desafio.md` antes de criar `feature/phase-03-videos` | `docs(desafio): consolida plano orquestrador autonomo e estado inicial da Fase 03` |
| **C1** | `feature/phase-03-videos` | Fim da Etapa `E1` (`research`) | `docs(decisions): mapeia opcoes e trade-offs arquiteturais da Fase 03 de videos` |
| **C2** | `feature/phase-03-videos` | Fim da Etapa `E2` (`plan-context` + 1º `plan-validate` `dirty`) | `docs(plan): consolida contexto herdado e validacao inicial da Fase 03` |
| **C3** | `feature/phase-03-videos` | Fim da Etapa `E3` (`plan-resolve` + `library-refs.md` + `plan-validate` `clean`) | `docs(plan): fixa decisoes tecnicas, referencias Context7 e validacao clean da Fase 03` |
| **C4** | `feature/phase-03-videos` | Fim da Etapa `E4` (`plan-build` + `plan-test-specs`) | `docs(plan): gera plano executavel phase-03-videos com SIs, contratos e specs E2E` |
| **C5..CN** | `feature/phase-03-videos` | **1 commit imediato ao final de CADA SI concluído** na Etapa `E5` (após testes do SI + `tsc` + `lint` verdes e `progress.md` atualizado) | `build(infra): ...` / `feat(storage): ...` / `feat(videos): ...` / `feat(worker): ...` / `chore(openapi): ...` *(um commit por SI real de `phase-03-videos.md`)* |
| **CF-doc** | `feature/phase-03-videos` | Fim da Etapa `E7` (paridade `AGENTS.md` + `CLAUDE.md` + `.agents/rules/` + C4) | `docs(agents): documenta modulo de videos, fila BullMQ, worker e storage em AGENTS e CLAUDE` |
| **CF-merge** | `feature/implementa-desafio-dev` | Fim da Etapa `E8` (`git merge --no-ff feature/phase-03-videos`) | `merge: integra Fase 03 de upload, processamento assincrono e streaming de videos em feature/implementa-desafio-dev` |

---

## 7. Roteiro Passo a Passo de Execução Completa (Etapas `E0` a `E8`)

---

### Passo Zero (Sempre Executar ao Iniciar ou Retomar) — Protocolo de Retomada

Sempre que iniciar uma sessão de execução deste plano, execute **antes de qualquer outra ação**:

1. Verifique se `docs/desafio/execucao-status.md` existe no disco (`read_file`).
2. **Caso NÃO exista (Execução Nova):** prossiga diretamente para a **Etapa `E0`**.
3. **Caso JÁ exista (Retomada de Execução Interrompida):**
   - Leia `docs/desafio/execucao-status.md` integralmente para identificar `current_stage` e `current_substep`.
   - Audite o estado físico real no WSL:
     ```powershell
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git branch --show-current
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git status -sb
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git log -n 5 --oneline
     ```
   - Confirme que `git rev-parse main` e `git rev-parse dev` coincidem com `immutable_refs` registrados em `execucao-status.md`.
   - Reconcilie o ponto exato de retomada:
     - Se `current_stage` for `E1..E4`: verifique os artefatos já escritos em `docs/decisions/` e `docs/phases/phase-03-videos/` (incluindo Gate 10 de sentinelas em `phase-03-videos.md` caso interrompido no meio do `plan-build`) e retome exatamente do sub-passo pendente.
     - Se `current_stage` for `E5`: leia `docs/phases/phase-03-videos/progress.md` (contrato de retomada nativo da skill `implement`), pule todos os SIs já marcados como `completed` e com commit registrado, suba os containers caso parados (`docker compose up -d`), e retome a partir do primeiro SI `pending` (concluindo eventuais arquivos parciais não commitados).
     - Se `current_stage` for `E6..E8`: retome diretamente da etapa indicada.
   - Registre uma linha no `## Diário de Checkpoints e Retomadas` de `docs/desafio/execucao-status.md` documentando a retomada.

---

### Etapa E0 — Pré-flight, Estado Inicial, Criação de `feature/phase-03-videos` e Baseline Verde

1. **E0.1 — Validar servidor MCP `context7_context7` (somente leitura):**
   - Leia `.agents/skills/` / skill `context7-mcp` e o schema de `context7_context7` (`resolve-library-id` e `query-docs`) e faça uma chamada rápida de teste via `call_mcp_tool` (ex.: `resolve-library-id` para `@nestjs/bullmq`). Se falhar por desconexão/ausência de MCP, pare e avise o usuário (jamais altere arquivos fora do projeto).
2. **E0.2 — Capturar SHAs imutáveis, limpar `filemode` espúrio e criar `docs/desafio/execucao-status.md`:**
   - Confirme no WSL que a branch atual é `feature/implementa-desafio-dev`, reverta qualquer diff de permissão em `scripts/sync-openapi.sh` (`git checkout -- scripts/sync-openapi.sh`) e capture os hashes de `main` e `dev` (`git rev-parse main` e `git rev-parse dev`).
   - Crie `docs/desafio/execucao-status.md` a partir do template da Seção 2.4 com `current_stage: E0` concluindo em transição para `E1`.
3. **E0.3 — Commitar setup em `feature/implementa-desafio-dev` (`C0`) e derivar `feature/phase-03-videos`:**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git branch --show-current
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/desafio/plano-implementacao-desafio.md docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(desafio): consolida plano orquestrador autonomo e estado inicial da Fase 03"
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git checkout -b feature/phase-03-videos
   ```
   *(Confirme que `git branch --show-current` agora retorna literalmente `feature/phase-03-videos`).*
4. **E0.4 — Subir infraestrutura atual e validar Baseline Verde das Fases 01–02:**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose up -d
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T db pg_isready -U streamtube
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm install
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run migration:run
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm test
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run test:e2e
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npx tsc --noEmit
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run lint
   ```

---

### Etapa E1 — Research: Decisões Técnicas da Fase 03 (`skill: research`)

1. **Ler integralmente `.agents/skills/research/SKILL.md`** e seguir rigorosamente seu procedimento (aplicando a precedência da Seção 2.1).
2. Ler `docs/project-plan.md` (Fase 03), `docs/decisions/technical-decisions-phase-01-configuracao-base.md`, `docs/decisions/technical-decisions-phase-02-auth.md`, `docs/decisions/technical-decisions-openapi-docs-nestjs.md` e `nestjs-project/package.json`.
3. Consultar o **Context7 MCP** (`context7_context7`: `resolve-library-id` + `query-docs`) para verificar compatibilidade com NestJS 11 / Node 25 / TypeORM 0.3 das opções avaliadas (`@nestjs/bullmq`, `bullmq`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, etc.).
4. Emitir `docs/decisions/technical-decisions-phase-03-videos.md` com:
   - Frontmatter `scope_type: phase`, `related_phases: [3]`, `status: pending`, `date`, `scope_description`.
   - `_Subprojects in scope:_` listando `nestjs-project/` e a justificativa explícita de não aplicabilidade para `next-frontend/` nesta fase (Coverage Check da skill `research`).
   - Todas as decisões estratégicas da Fase 03 (fila, storage/buckets/chaves, upload de até 10GB + rascunho, worker/container, processamento FFmpeg/thumbnail, URL única, streaming/download e ciclo de status/falhas), cada uma rastreada via `**Capability:**` literal aos bullets de `docs/project-plan.md` e com `**Decision:** _[pending]_`!
5. Atualizar `docs/desafio/execucao-status.md` (`E0` e `E1` `completed`, `current_stage: E2`), validar `git branch --show-current == feature/phase-03-videos` e commitar (**C1**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/decisions/technical-decisions-phase-03-videos.md docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(decisions): mapeia opcoes e trade-offs arquiteturais da Fase 03 de videos"
   ```

---

### Etapa E2 — Planejamento Parte 1: Contexto e 1ª Validação (`plan-context` + `plan-validate`)

1. **Ler `.agents/skills/plan-pipeline/SKILL.md`, `.agents/skills/plan-context/SKILL.md` e `.agents/skills/plan-context/references/context-output-template.md`.**
2. Ler `.agents/agents/{plan-reader,decisions-reader,decisions-detail-reader,decisions-correlator,phases-reader}.md`, registrá-los via `define_subagent` (`EnableWriteTools: false`) e invocá-los em paralelo via `invoke_subagent` (`Workspace: "inherit"`).
3. Executar a verificação obrigatória do `Filter Trace` (Step 2 de `plan-context`), incluir `openapi-docs-nestjs` nas decisões correlacionadas herdadas, ler `.agents/skills/testing-guide-nestjs-project/SKILL.md`, capturar `sources_mtime` reais (`stat -c '%y'`) e gravar **`docs/phases/phase-03-videos/context.md`** (`kind: phase`, `name: phase-03-videos`).
4. **Ler `.agents/skills/plan-validate/SKILL.md`** e executar a 1ª passagem sobre `context.md`, gerando **`docs/phases/phase-03-videos/validation.md`** (`status: dirty` com as questões `OQ-N` dos TDs pendentes).
   - *Se `plan-validate` apontar qualquer `MD-N` (capability sem cobertura ou decisão estratégica faltante):* volte ao `research` para adicionar o TD correspondente em `technical-decisions-phase-03-videos.md`, regenere `context.md` e rode `plan-validate` novamente antes de seguir!
5. Atualizar `docs/desafio/execucao-status.md` (`E2` `completed`, `current_stage: E3`), validar branch `feature/phase-03-videos` e commitar (**C2**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/phases/phase-03-videos/context.md docs/phases/phase-03-videos/validation.md docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): consolida contexto herdado e validacao inicial da Fase 03"
   ```

---

### Etapa E3 — Planejamento Parte 2: Resolução, `library-refs.md` via Context7 e Validação `clean` (`plan-resolve` + `plan-validate`)

1. **Ler `.agents/skills/plan-resolve/SKILL.md` integralmente.**
2. Aplicar as decisões da Seção 5.1 em `docs/decisions/technical-decisions-phase-03-videos.md` (`status: decided`, campos `**Decision:**` e `**Libraries:**` preenchidos, tabela `## Decisions Summary` atualizada).
3. Executar o Step 5 de `plan-resolve`: re-disparar `decisions-reader` e `decisions-detail-reader` (validando `Filter Trace` e contagem exata de TDs decididos), atualizar `## Decisions Index`, `## Decisions Detail` e `sources_mtime` em `context.md`, mover as issues resolvidas para `## Resolved Issues` em `validation.md` (`resolved_by: phase-03-videos/TD-XX`), e consultar o **Context7 MCP** (`context7_context7`: `resolve-library-id` + `query-docs`) para materializar **`docs/phases/phase-03-videos/library-refs.md`** cobrindo todas as bibliotecas novas decididas (`@nestjs/bullmq`, `bullmq`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`) com `sources_mtime` real.
4. **Re-executar `plan-validate` (`2ª passagem`):**
   - Confirmar que `docs/phases/phase-03-videos/validation.md` atinge **`status: clean`** e **`issue_count: 0`**, mantendo `## Resolved Issues` intacto como trilha de auditoria!
   - Caso reste qualquer issue aberta, itere `plan-resolve ↔ plan-validate` até fechar 100% em `status: clean`.
5. Atualizar `docs/desafio/execucao-status.md` (`E3` `completed`, `current_stage: E4`), validar branch `feature/phase-03-videos` e commitar (**C3**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/decisions/technical-decisions-phase-03-videos.md docs/phases/phase-03-videos/context.md docs/phases/phase-03-videos/validation.md docs/phases/phase-03-videos/library-refs.md docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): fixa decisoes tecnicas, referencias Context7 e validacao clean da Fase 03"
   ```

---

### Etapa E4 — Planejamento Parte 3: Plano Executável e Test Specs (`plan-build` + `plan-test-specs`)

1. **Ler `.agents/skills/plan-build/SKILL.md`, `.agents/skills/plan-build/phase-a.md`, `nestjs-project/CLAUDE.md` e os 5 templates em `.agents/skills/plan-build/templates/tech-specs/{data-model,api-contracts,auth-matrix,error-catalog,events-messages}.md`.**
2. Executar **Phase A** do `plan-build`:
   - Passar todos os Gates 1–10.
   - Escrever o scaffold de `docs/phases/phase-03-videos/phase-03-videos.md` (`kind: phase`, `name: phase-03-videos`, `test_specs_aware: true`, `sources_mtime` real).
   - Inserir via chamadas `edit_file` incrementais as 5 subseções de `## Technical Specifications` (`### Data Model`, `### API Contracts`, `### Authorization Matrix`, `### Error Catalog`, `### Events/Messages`), respeitando o *Field-name verbatim invariant* (`A4`) e gravando `<!-- phase-a-complete -->` (`A4.6`).
3. Prosseguir na mesma sessão (`A5` → *Continue now*) lendo **`.agents/skills/plan-build/phase-b.md`** e executando **Phase B**:
   - Decompor o escopo em Step Implementations (`SI-03.1`, `SI-03.2`, ...) cobrindo integralmente a superfície técnica da Seção 5.2 e os cuidados de regressão da Seção 4.3, respeitando rigorosamente todas as regras de `phase-b.md`:
     - Tetos rígidos de `B3`: `≤ 5 Technical actions`, `≤ 5 Test files` e `≤ 10 Acceptance criteria` por SI (aplicando auto-split se necessário).
     - Emissão condicional de `**Route:** <METHOD> /...` + `**Test Specs:** _pending /plan-test-specs_` nos SIs plain de controller wiring backend (omitindo linhas E2E da tabela `Tests` inline nesses SIs, conforme `phase-b.md`).
     - Nota arquitetural importante para ordem de SIs: só adicionar o serviço `video-worker` ao `compose.yaml` no SI que já entrega `src/worker.main.ts` (evitando crash-loop de container antes do entrypoint existir!).
   - Inserir **um SI por `edit_file`** (`B4`), preencher `## Dependency Map` (`B5`) e `## Deliverables` (`B6` com comandos `docker compose exec nestjs-api ...`), removendo `<!-- phase-a-complete -->`.
4. **Ler `.agents/skills/plan-test-specs/SKILL.md`** e executar `plan-test-specs`:
   - Se `phase-03-videos.md` contiver placeholders `**Test Specs:** _pending /plan-test-specs_`, gerar os arquivos `nestjs-project/specs/<scenario>.plan.md` (`subproject: backend`, `runner: jest+supertest`, `target_file: test/<feature>.e2e-spec.ts`) e atualizar os campos `**Test Specs:** see \`nestjs-project/specs/...\`` em `phase-03-videos.md`. Se `TEST_SPECS_COUNT == 0`, registrar o skip silencioso previsto pela skill.
5. Atualizar `docs/desafio/execucao-status.md` (`E4` `completed`, populando a seção `## Progresso Granular de SIs na Etapa E5` com os IDs exatos de `phase-03-videos.md`, `current_stage: E5`), validar branch `feature/phase-03-videos` e commitar (**C4**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/phases/phase-03-videos/ docs/desafio/execucao-status.md nestjs-project/specs/
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): gera plano executavel phase-03-videos com SIs, contratos e specs E2E"
   ```
   *(Se `nestjs-project/specs/` não tiver sido criado por ausência de placeholders, omita `nestjs-project/specs/` do `git add`).*

---

### Etapa E5 — Implementação Passo a Passo Guiada pelo Plano (`skill: implement`)

1. **Ler `.agents/skills/implement/SKILL.md` integralmente** (operando em modo `continuous`), além de `.agents/skills/nestjs-best-practices/SKILL.md`, `.agents/skills/typeorm/SKILL.md`, `.agents/skills/testing-guide-nestjs-project/SKILL.md` (e sub-guias `artifacts/*.md` / `references/*.md` sob demanda) e `.agents/rules/nestjs-*.md` / `typeorm-*.md` / `typescript-strict.md`.
2. Executar o **Preflight do `implement`**:
   - Confirmar branch `feature/phase-03-videos`.
   - Indexar cabeçalhos de `docs/phases/phase-03-videos/phase-03-videos.md` sem carregar o arquivo inteiro de uma vez (seguindo a leitura sob demanda por SI de `implement/SKILL.md`).
   - Executar o **Test Specs preflight** (`PLAN_MODE=modern`): zero `_pending_` e zero specs `MISSING`.
   - Criar `docs/phases/phase-03-videos/progress.md` com todos os SIs de `phase-03-videos.md` como `pending` (ou retomar do primeiro `pending` se já existir).
3. **Loop Estrito por SI (em ordem topológica do `Dependency Map` de `phase-03-videos.md`):**
   Para **cada SI** em `docs/phases/phase-03-videos/phase-03-videos.md`:
   - **Passo 1 (Plan the SI):** Ler apenas o bloco do SI atual em `phase-03-videos.md`, as subseções referenciadas de Tech Specs e a seção relevante de `library-refs.md`.
   - **Passo 2 (Implement):** Executar as ações técnicas dentro do escopo estrito do SI, garantindo os requisitos transversais descobertos no código:
     - Infraestrutura real no `compose.yaml`: `minio` (`9000`/`9001`), `redis` (`6379`), `ffmpeg` no `Dockerfile.dev` e `video-worker` (`npm run start:worker:dev`, sem porta HTTP).
     - Quando criar a relação `Video ↔ Channel` e a migration `CreateVideos` (gerada exclusivamente via `docker compose exec -T nestjs-api npm run migration:generate -- src/database/migrations/CreateVideos`), atualizar simultaneamente `cleanAllTables()` em `src/test/create-test-data-source.ts` e os **11 arquivos** `ALL_ENTITIES` + `migrations.integration-spec.ts` (§ 4.3 item 4).
     - Upload de até `10 GiB` via S3 Presigned Multipart Upload sem jamais passar bytes do vídeo pela API NestJS; pré-cadastro imediato em `status = 'draft'`; `short_code` único de 11 chars com retry `23505`.
     - Worker FFmpeg consumindo Presigned GET URL via HTTP `Range` (`execFile` sem shell), gerando thumbnail JPEG no MinIO, atualizando `duration_seconds`, `width`, `height`, `video_codec`, `thumbnail_key` e `status = 'ready'`, e tratando falhas com `attempts: 3` exponencial + `UnrecoverableError` + `@OnWorkerEvent('failed')` persistindo `status = 'failed'` sem derrubar o processo.
     - Streaming `206 Partial Content` (`Range` RFC 9110 + `416` + `200` fallback) e Download (`Content-Disposition: attachment`) não-bufferizados via `StreamableFile`, decorados com `@Public()`, `@SkipThrottle()` e `@nestjs/swagger` completo (`ApiErrorEnvelope`), exportando `nestjs-project/openapi.json` atualizado via `npm run openapi:export`.
   - **Passo 3 & 3a (Write Tests + JIT Spec Read):** Se o SI tiver `**Test Specs:** see ...`, ler o `.plan.md` e autorar o arquivo `test/*.e2e-spec.ts` correspondente (Step 3a); criar os arquivos de teste Unit (`*.spec.ts`) e Integration (`*.integration-spec.ts` contra PostgreSQL, MinIO, Redis e FFmpeg reais, usando `Buffer` em memória para a fixture `.mp4` sem escrever fora do repositório!).
   - **Passo 4 & 5 (Run Tests + Fix Loop até 3x):** Executar dentro do container `nestjs-api`:
     - Os testes do SI (`docker compose exec -T nestjs-api npx jest ... --runInBand` e/ou `npm run test:e2e -- ...`)
     - `docker compose exec -T nestjs-api npx tsc --noEmit` (deve sair com `0`!)
     - `docker compose exec -T nestjs-api npm run lint` (deve sair com `0`!)
     - *Só avance quando 100% passarem!*
   - **Passo 6 (Persistir Progresso + Commit Git Flow Atômico do SI):**
     - Atualizar `docs/phases/phase-03-videos/progress.md` (status `completed`, contagem real de testes passando e observações reais).
     - Atualizar `docs/desafio/execucao-status.md` marcando o SI concluído.
     - Verificar `git branch --show-current == feature/phase-03-videos` e `git status -sb` (apenas arquivos dentro do repositório e do escopo do SI) e realizar imediatamente o **commit semântico do SI**!
4. Ao concluir o último SI, marcar `**Status:** completed` em `docs/phases/phase-03-videos/progress.md` e rodar a suíte completa (`npm test` + `npm run test:e2e` + `npx tsc --noEmit` + `npm run build` + `npm run lint`).

---

### Etapa E6 — Verificação Empírica Ponta a Ponta em Runtime Real (Docker Compose)

1. Verificar todos os 6 containers ativos via `docker compose ps` e os healthchecks da Seção 4.2 (`db`, `redis`, `minio`, `ffmpeg`, `video-worker`).
2. Compilar a aplicação (`docker compose exec -T nestjs-api npm run build`) e executar um smoke test ponta a ponta real em uma única chamada controlada dentro do container (subindo `node dist/main` temporariamente, executando cadastro → confirmação → login → `POST /videos/uploads` → `PUT` de vídeo `.mp4` real para o MinIO → `POST /videos/:id/complete` → aguardando o container `video-worker` processar até `status === 'ready'` → validando `206 Partial Content` com `Range: bytes=0-511` em `/stream`, `200` `image/jpeg` em `/thumbnail` e `Content-Disposition: attachment` em `/download`, limpando os dados de smoke test no banco/MinIO e encerrando o processo HTTP temporário ao final).
3. Registrar as evidências observadas no diário de `docs/desafio/execucao-status.md` (`E6` `completed`, `current_stage: E7`).

---

### Etapa E7 — Atualização e Paridade da Documentação de IA

1. Atualizar com **100% de fidelidade ao código real** e em paridade dupla (Antigravity + Claude Code):
   - `AGENTS.md` e `CLAUDE.md` (raiz): atualizar `Repository Structure`, trocar `Message Queue (TBD)` por `Message Queue (Redis / BullMQ)` em `Architecture`, e adicionar a seção de vídeos descrevendo fielmente módulos, fluxo de upload de até 10GB, fila/worker, streaming/download e endpoints reais.
   - `.agents/rules/nestjs-project.md`, `nestjs-project/CLAUDE.md` e `nestjs-project/AGENTS.md`: documentar os 6 serviços do `compose.yaml`, probes de readiness (`db`, `redis`, `minio`, `ffmpeg`, `video-worker`), scripts do worker e cuidados de teste.
   - `docs/diagrams/software-arch.mermaid`: atualizar `Message Queue` de `"TBD"` para `"Redis / BullMQ"`.
2. **Auditoria Anti-Alucinação:** Conferir no disco/código que **todo** caminho de arquivo, script `npm`, serviço Docker, variável `.env` e rota HTTP citada nos documentos acima existe literalmente com o mesmo nome (*"Documentação que cite arquivos ou comportamentos inexistentes reprova"*).
3. Atualizar `docs/desafio/execucao-status.md` (`E7` `completed`, `current_stage: E8`), validar branch `feature/phase-03-videos` e commitar (**CF-doc**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add AGENTS.md CLAUDE.md nestjs-project/AGENTS.md nestjs-project/CLAUDE.md .agents/rules/nestjs-project.md docs/diagrams/software-arch.mermaid docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(agents): documenta modulo de videos, fila BullMQ, worker e storage em AGENTS e CLAUDE"
   ```

---

### Etapa E8 — Auditoria Final de Definition of Done, Anti-Reprova e Reintegração na Branch Base

1. Executar sequencialmente todos os 7 comandos da **Definition of Done (§ 4.4)** dentro do container `nestjs-api` e confirmar código de saída `0` em todos e `git status` limpo.
2. Conferir todos os itens da **Seção 8 (Checklist Final de Aceite e Anti-Reprova)**.
3. Atualizar `docs/desafio/execucao-status.md` marcando `E8: completed` e `overall_status: completed`, commitar na `feature/phase-03-videos`, fazer checkout para a **branch base `feature/implementa-desafio-dev`** e realizar o merge `--no-ff` (**CF-merge**):
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/desafio/execucao-status.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(desafio): marca execucao completa da Fase 03 como concluida e auditada"
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git checkout feature/implementa-desafio-dev
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git merge --no-ff feature/phase-03-videos -m "merge: integra Fase 03 de upload, processamento assincrono e streaming de videos em feature/implementa-desafio-dev"
   ```
4. **Auditoria Final de Confinamento:**
   - Executar `git rev-parse main dev` e comprovar que ambos os hashes são **idênticos** aos capturados em `immutable_refs` na Etapa `E0` (nenhum commit nem alteração em `main`, `dev` ou qualquer outra branch!).
   - Executar `git status -sb` e comprovar árvore de trabalho 100% limpa em `feature/implementa-desafio-dev`.

---

## 8. Checklist Final de Aceite, Rastreabilidade e Anti-Reprova Automática

- [ ] **Confinamento de arquivos:** Nenhum arquivo fora de `./mba-ia-greenfield-project/` foi criado, editado ou removido.
- [ ] **Confinamento de branches:** Somente `feature/implementa-desafio-dev` e `feature/phase-03-videos` receberam commits; `main` e `dev` permanecem com seus SHAs originais intactos.
- [ ] **Controle de retomada:** `docs/desafio/execucao-status.md` presente, consistente com `git log` e com `overall_status: completed`.
- [ ] **Precedência das skills e rastreabilidade de IA:**
  - [ ] `docs/decisions/technical-decisions-phase-03-videos.md` com `status: decided` e todas as decisões em aberto justificadas e rastreadas aos bullets de `docs/project-plan.md`.
  - [ ] `docs/phases/phase-03-videos/context.md` consolidado com 100% de cobertura das 9 capabilities da Fase 03.
  - [ ] `docs/phases/phase-03-videos/validation.md` fechado em **`status: clean`** (`issue_count: 0`) e preservando o histórico auditável em `## Resolved Issues`.
  - [ ] `docs/phases/phase-03-videos/library-refs.md` preenchido via consultas reais ao servidor MCP Context7 (`context7_context7`).
  - [ ] `docs/phases/phase-03-videos/phase-03-videos.md` contendo `Step Implementations` (`SI-03.x`), `Technical Specifications` (`Data Model`, `API Contracts`, `Authorization Matrix`, `Error Catalog`, `Events/Messages`), `Dependency Map` e `Deliverables` (+ `nestjs-project/specs/*.plan.md` se SIs emitiram `**Test Specs:**`).
  - [ ] `docs/phases/phase-03-videos/progress.md` com `Status: completed` e resultados reais por SI.
- [ ] **Funcionalidades da Fase 03:** Upload direto ao storage de até 10 GiB sem travar a API + pré-cadastro em rascunho (`draft`) + processamento automático no worker (`ffprobe` metadados/duração + `ffmpeg` thumbnail) + URL única (`short_code`) sem conflito + streaming HTTP `Range` (`206 Partial Content`) + download (`attachment`) + ciclo `draft → processing → ready | failed` persistido no PostgreSQL.
- [ ] **Infraestrutura e Qualidade (Definition of Done):** `minio`, `redis` e `video-worker` subindo junto com `nestjs-api`, `db` e `mailpit` no `compose.yaml`; migration `CreateVideos` aplicada; `npm test`, `npm run test:e2e`, `npx tsc --noEmit`, `npm run build` e `npm run lint` todos verdes (exit `0` dentro do container); `openapi.json` sincronizado.
- [ ] **Documentação de IA coerente:** `AGENTS.md`, `CLAUDE.md`, `nestjs-project/AGENTS.md`, `nestjs-project/CLAUDE.md`, `.agents/rules/nestjs-project.md` e `docs/diagrams/software-arch.mermaid` atualizados e 100% verificados contra os arquivos e rotas reais do código.
