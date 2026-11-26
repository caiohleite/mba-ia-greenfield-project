# Plano Mestre de Implementação — Desafio Fase 03: Upload e Processamento de Vídeos (StreamTube)

> **Agente Executor Alvo:** Google Antigravity  
> **Branch Base de Trabalho:** `feature/implementa-desafio-dev` (já deriva de `dev` e recebeu merge de `main`; a nova branch `feature/phase-03-videos` deriva diretamente dela e é reintegrada nela)  
> **Escopo:** Fase 03 completa (Backend NestJS 11 + Object Storage S3/MinIO + Fila Redis/BullMQ + Video Worker FFmpeg + Artefatos do Pipeline de IA + Testes Unit/Integration/E2E + Documentação de IA)  
> **Fora de Escopo:** Interface de vídeo no `next-frontend/` (pertence às Fases 04 e 05 conforme `docs/project-plan.md` e `docs/desafio/desafio.md`)  
> **Referência Normativa:** `docs/desafio/desafio.md`, `docs/project-plan.md`, `AGENTS.md`, `CLAUDE.md` e `.agents/`

---

## Sumário

1. [Visão Geral e Contrato de Entregáveis](#1-visão-geral-e-contrato-de-entregáveis)
2. [Operação Nativa no Google Antigravity (Skills, Subagents, MCP e Regras)](#2-operação-nativa-no-google-antigravity-skills-subagents-mcp-e-regras)
3. [Critérios Obrigatórios para Executar e Testar a Aplicação](#3-critérios-obrigatórios-para-executar-e-testar-a-aplicação)
4. [Decisões Arquiteturais da Fase 03 (`TD-01` a `TD-08`) e Especificação Alvo](#4-decisões-arquiteturais-da-fase-03-td-01-a-td-08-e-especificação-alvo)
5. [Governança Git Flow e Mapa Granular de Commits](#5-governança-git-flow-e-mapa-granular-de-commits)
6. [Roteiro Passo a Passo de Execução Completa (Etapas 0 a 8)](#6-roteiro-passo-a-passo-de-execução-completa-etapas-0-a-8)
   - [Etapa 0 — Pré-flight, Higienização Git, Criação da Branch da Fase 03 e Baseline Verde](#etapa-0--pré-flight-higienização-git-criação-da-branch-da-fase-03-e-baseline-verde)
   - [Etapa 1 — Research: Documento de Decisões Técnicas (`research`)](#etapa-1--research-documento-de-decisões-técnicas-research)
   - [Etapa 2 — Planejamento Parte 1: Consolidação de Contexto e Validação Inicial (`plan-context` + `plan-validate`)](#etapa-2--planejamento-parte-1-consolidação-de-contexto-e-validação-inicial-plan-context--plan-validate)
   - [Etapa 3 — Planejamento Parte 2: Resolução de Decisões, Context7 (`library-refs.md`) e Validação `clean` (`plan-resolve` + `plan-validate`)](#etapa-3--planejamento-parte-2-resolução-de-decisões-context7-library-refsmd-e-validação-clean-plan-resolve--plan-validate)
   - [Etapa 4 — Planejamento Parte 3: Construção do Plano Executável e Test Specs (`plan-build` + `plan-test-specs`)](#etapa-4--planejamento-parte-3-construção-do-plano-executável-e-test-specs-plan-build--plan-test-specs)
   - [Etapa 5 — Implementação Guiada Passo a Passo (`implement`: `SI-03.1` a `SI-03.10`)](#etapa-5--implementação-guiada-passo-a-passo-implement-si-031-a-si-0310)
   - [Etapa 6 — Verificação Ponta a Ponta em Runtime Real (Docker Compose Completo)](#etapa-6--verificação-ponta-a-ponta-em-runtime-real-docker-compose-completo)
   - [Etapa 7 — Atualização e Paridade da Documentação de IA](#etapa-7--atualização-e-paridade-da-documentação-de-ia)
   - [Etapa 8 — Auditoria Final de Definition of Done, Anti-Reprova e Merge na Branch Base (`feature/implementa-desafio-dev`)](#etapa-8--auditoria-final-de-definition-of-done-anti-reprova-e-merge-na-branch-base-featureimplementa-desafio-dev)
7. [Checklist Final de Aceite e Blindagem contra Reprova Automática](#7-checklist-final-de-aceite-e-blindagem-contra-reprova-automática)

---

## 1. Visão Geral e Contrato de Entregáveis

O objetivo deste plano é conduzir o agente **Antigravity** na execução integral e irretocável da **Fase 03 — Upload e Processamento de Vídeos** do projeto **StreamTube**, respeitando rigorosamente o pipeline de engenharia orientado por IA do repositório (`research → plan-context → plan-validate → plan-resolve → plan-validate [clean] → plan-build → [plan-test-specs] → implement`).

Ao final da execução deste plano, o repositório conterá os seguintes artefatos rastreáveis entre si e 100% funcionais:

```text
mba-ia-greenfield-project/
├── AGENTS.md                                                    ← atualizado com arquitetura real da Fase 03
├── CLAUDE.md                                                    ← atualizado em paridade com AGENTS.md
├── .agents/
│   └── rules/
│       └── nestjs-project.md                                    ← atualizado com novos serviços Compose e worker
├── docs/
│   ├── desafio/
│   │   ├── desafio.md                                           ← enunciado oficial do desafio
│   │   ├── relatorio-adaptacao-antigravity.md                   ← relatório da migração Claude Code → Antigravity
│   │   └── plano-implementacao-desafio.md                       ← este plano mestre de execução
│   ├── diagrams/
│   │   └── software-arch.mermaid                                ← atualizado (Message Queue: Redis / BullMQ)
│   ├── decisions/
│   │   └── technical-decisions-phase-03-videos.md               ← gerado via skill `research` e decidido via `plan-resolve`
│   └── phases/
│       └── phase-03-videos/                                     ← pasta canônica da Fase 03
│           ├── context.md                                       ← gerado via skill `plan-context`
│           ├── validation.md                                    ← gerado via skill `plan-validate` (`status: clean`)
│           ├── library-refs.md                                  ← gerado via skill `plan-resolve` (Context7 MCP)
│           ├── phase-03-videos.md                               ← gerado via skill `plan-build` (SIs + Tech Specs + Dep Map + Deliverables)
│           └── progress.md                                      ← mantido via skill `implement` (`Status: completed`)
└── nestjs-project/
    ├── AGENTS.md                                                ← criado/sincronizado para descoberta hierárquica no Antigravity
    ├── CLAUDE.md                                                ← atualizado (lido por `plan-build` e `implement`)
    ├── Dockerfile.dev                                           ← atualizado com `ffmpeg` (inclui `ffmpeg` e `ffprobe`)
    ├── compose.yaml                                             ← `nestjs-api`, `db`, `mailpit` + `minio`, `redis`, `video-worker`
    ├── .env.example                                             ← atualizado com variáveis de Storage, Redis/Queue e Upload
    ├── package.json                                             ← scripts do worker + dependências fixadas
    ├── openapi.json                                             ← regenerado via `npm run openapi:export` com rotas de vídeos
    ├── specs/                                                   ← specs E2E gerados por `plan-test-specs` (se aplicável)
    ├── src/
    │   ├── config/
    │   │   ├── storage.config.ts                                ← `registerAs('storage', ...)`
    │   │   ├── queue.config.ts                                  ← `registerAs('queue', ...)`
    │   │   └── env.validation.ts                                ← schema Joi expandido
    │   ├── storage/                                             ← módulo S3/MinIO (`StorageModule`, `StorageService`)
    │   ├── videos/                                              ← módulo de domínio (`VideosModule`, `VideosController`, `VideosService`, DTOs, `Video` entity, `VideoProcessor`, `MediaProbeService`)
    │   ├── worker/                                              ← módulo raiz do processo worker (`WorkerModule`) + entrypoint `src/worker.main.ts`
    │   └── database/migrations/
    │       └── <timestamp>-CreateVideos.ts                      ← migration gerada via TypeORM CLI
    └── test/
        └── videos.e2e-spec.ts                                   ← suíte E2E completa contra PostgreSQL + MinIO + Redis + Worker reais
```

---

## 2. Operação Nativa no Google Antigravity (Skills, Subagents, MCP e Regras)

A fundação de IA do projeto já está migrada em `.agents/` e `AGENTS.md` (conforme documentado em `docs/desafio/relatorio-adaptacao-antigravity.md`). Para que todas as skills do pipeline funcionem sem desvios no Antigravity, o agente executor **DEVE** aplicar a tabela de tradução operacional abaixo sempre que uma skill mencionar primitivas originais do Claude Code:

### 2.1. Tabela de Tradução de Ferramentas (Claude Code → Antigravity)

| Primitiva citada na Skill | Equivalente Exato no Antigravity | Regra de Uso Obrigatória |
| :--- | :--- | :--- |
| `Skill` tool / `/research`, `/plan-context`, etc. | `read_file` no arquivo `.agents/skills/<skill>/SKILL.md` (e arquivos companheiros citados, ex.: `phase-a.md`, `phase-b.md`, `references/*`, `templates/*`) | **Sempre** ler o `SKILL.md` correspondente antes de executar qualquer etapa do pipeline ou de codificação (`nestjs-best-practices`, `typeorm`, `testing-guide-nestjs-project`). |
| `AskUserQuestion` | Ferramenta `ask_question` | Usar `ask_question` para bateladas de escolhas estruturadas (ex.: `plan-resolve`, confirmação de correlação em `plan-context`, pausa `A5` em `plan-build`). Em modo autônomo autorizado pelo usuário, seguir as opções recomendadas na Seção 4 deste plano se o usuário instruir execução direta sem pausas. |
| `Agent` com `subagent_type` customizado (`plan-reader`, `decisions-reader`, `decisions-detail-reader`, `decisions-correlator`, `phases-reader`, `inventory-digest-reader`) | `define_subagent` + `invoke_subagent` | Como subagents em `.agents/agents/*.md` não ficam pré-registrados na lista global, ler `.agents/agents/<nome>.md` via `read_file`, registrá-lo na conversa com `define_subagent` (`Name: "<nome>"`, `EnableWriteTools: false`, `SystemPrompt`: corpo do arquivo `.md`) e dispará-los em paralelo via `invoke_subagent`. Encerrar o turno sem polling para receber as respostas via wakeup reativo. |
| `Agent` com `subagent_type: Explore` | `invoke_subagent` com `TypeName: "research"` | Passar no `Prompt` o contrato estrito de resposta enxuta exigido por `implement/SKILL.md` (assinaturas/padrões, nunca arquivos inteiros) e ponteiros explícitos para `.agents/rules/*.md` quando necessário. |
| `Agent` com `subagent_type: general-purpose` (usado em `implement` Step 4 para rodar testes) | `invoke_subagent` com `TypeName: "self"` (ou execução direta via `run_command` quando mais enxuto) | Se delegar ao `self`, instruir explicitamente no prompt a ler `nestjs-project/CLAUDE.md` e `.agents/rules/nestjs-project.md` e executar comandos via WSL (`BypassSandbox: true`, flag `-T`), retornando relatório diagnóstico sem ruído. |
| `TaskCreate` / `TaskUpdate` (lista persistente de SIs em `implement`) | Artefato `task.md` em `<appDataDir>\brain\<conversation-id>\task.md` (`write_file`/`edit_file` com `ArtifactMetadata`) + `docs/phases/phase-03-videos/progress.md` | Criar 1 item por SI em `task.md` antes do loop de implementação e atualizar em sincronia estrita com `progress.md` a cada SI concluído. |
| `Read`, `Write`, `Edit`, `Glob`, `Grep` | `read_file`, `write_file`, `edit_file`, `run_command` (PowerShell / `rg` / `Get-ChildItem` / `Select-String`) | Para leituras delimitadas exigidas pelas skills (`offset` + `limit`), usar `StartLine` e `EndLine` em `read_file`. |
| `mcp__context7__resolve-library-id` e `mcp__context7__query-docs` | Ferramentas MCP do servidor **Context7** habilitado no Antigravity | **Obrigatório** consultar antes de fixar libs em `research`/`plan-resolve` e antes de implementar integrações com bibliotecas em `implement`. Ver verificação na Etapa 0. |

### 2.2. Armadilha Crítica de Nomenclatura de Slug (`phase-03-videos`)

Atenção máxima a esta regra de compatibilidade entre `docs/desafio/desafio.md` e as skills de `plan-pipeline`:

1. O enunciado (`docs/desafio/desafio.md`, seções *O workflow do projeto*, *Requisitos*, *Critérios de Aceite* e *Estrutura do entregável*) exige **literalmente** estes caminhos:
   - Documento de decisões: `docs/decisions/technical-decisions-phase-03-videos.md`
   - Pasta da fase: `docs/phases/phase-03-videos/`
   - Plano executável: `docs/phases/phase-03-videos/phase-03-videos.md`
2. Esse padrão espelha exatamente as fases existentes no repositório:
   - `docs/decisions/technical-decisions-phase-01-configuracao-base.md` → `docs/phases/phase-01-configuracao-base/phase-01-configuracao-base.md`
   - `docs/decisions/technical-decisions-phase-02-auth.md` → `docs/phases/phase-02-auth/phase-02-auth.md`
   - `docs/decisions/technical-decisions-phase-02-auth-frontend.md` → `docs/phases/phase-02-auth-frontend/phase-02-auth-frontend.md`
3. **Regra de Ouro:**
   - Ao executar `research`, salve **obrigatoriamente** em `docs/decisions/technical-decisions-phase-03-videos.md` (não omita `phase-03-` do nome do arquivo).
   - Os identificadores de referência das decisões (`Ref`) gerados por `decisions-reader` e `decisions-detail-reader` serão `phase-03-videos/TD-01`, `phase-03-videos/TD-02`, etc. (idênticos a `phase-02-auth/TD-01`).
   - Ao resolver `{target_dir}` e `{name}` em `plan-context`, `plan-validate`, `plan-resolve`, `plan-build`, `plan-test-specs` e `implement`, **NUNCA** duplique o prefixo `phase-03-`! Como o identificador extraído do arquivo já começa com `phase-03-`, o diretório alvo é **`docs/phases/phase-03-videos/`** (jamais `docs/phases/phase-03-phase-03-videos/`), o campo frontmatter é **`name: phase-03-videos`**, e o plano é **`docs/phases/phase-03-videos/phase-03-videos.md`**.

---

## 3. Critérios Obrigatórios para Executar e Testar a Aplicação

Antes de iniciar qualquer etapa, todos os critérios técnicos abaixo — verificados empiricamente neste ambiente e no código-fonte de `nestjs-project/` — devem ser rigorosamente obedecidos durante toda a execução:

### 3.1. Execução de Comandos Docker/Container no Ambiente Windows + WSL2

1. O workspace reside no WSL2 Ubuntu (`\\wsl.localhost\Ubuntu\home\c\FullCycle\desafios\mba-ia-greenfield-project` no Windows; `/home/c/FullCycle/desafios/mba-ia-greenfield-project` no Linux).
2. O binário `docker` **não existe no PowerShell do Windows** — o Docker Engine (`28.1.1`) e o Docker Compose (`v2.35.1`) rodam **dentro da distro WSL `Ubuntu`**.
3. O sandbox padrão do terminal Windows no Antigravity (`BypassSandbox: false`) bloqueia chamadas ao `wsl.exe` com `Wsl/E_ACCESSDENIED`. Portanto:
   - Operações locais de leitura/busca/git simples que funcionam em PowerShell rodam primeiro com `BypassSandbox: false`.
   - **Todo comando `docker compose`** (subir/descer containers, healthchecks, instalar pacotes, rodar migrations, testes, `tsc` e `lint`) **DEVE** ser executado invocando `wsl` diretamente com `BypassSandbox: true`, sem encadeamentos desnecessários, preservando um prefixo limpo e auto-aprovável:
     ```powershell
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose <subcomando>
     ```
   - **Sempre inclua a flag `-T`** em `docker compose exec -T nestjs-api <comando>` ao executar via `run_command`, evitando falha de alocação de pseudo-TTY (`the input device is not a TTY`).
4. Para capturar timestamps ISO-8601 exigidos em `sources_mtime` sem precisar sair do sandbox, pode-se usar PowerShell puro (`(Get-Item "<arquivo>").LastWriteTime.ToString("yyyy-MM-ddTHH:mm:sszzz")`) ou `wsl -d Ubuntu -- stat -c '%y' <arquivo>`. Mantenha um único formato consistente ao comparar `sources_mtime` entre estágios.

### 3.2. Regra Estrita Container-Only e Rede Docker

1. **Container-Only (`nestjs-project/CLAUDE.md` & `.agents/rules/nestjs-project.md`):** Todo comando `npm`, `npx`, `node`, `tsc`, `typeorm` e `jest` **DEVE** rodar dentro do container `nestjs-api`:
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm test
   ```
   Jamais rode `npm install`, `npm test` ou `npx tsc` diretamente no host Windows/WSL fora do container.
2. **Docker Networking (`AGENTS.md` / `CLAUDE.md`):** Toda comunicação entre containers usa **exclusivamente o nome do serviço no `compose.yaml`**, nunca `localhost` ou `127.0.0.1`:
   - PostgreSQL: `DB_HOST=db` (`5432`)
   - Mailpit: `MAIL_HOST=mailpit` (`1025` SMTP / `8025` HTTP API)
   - Redis (Fila): `REDIS_HOST=redis` (`6379`)
   - MinIO (Object Storage): `STORAGE_ENDPOINT=http://minio:9000` (API S3 `9000` / Console `9001`)
3. **Servidor HTTP em Desenvolvimento:** Subir o ambiente (`docker compose up -d`) significa subir os containers da stack (`db`, `mailpit`, `minio`, `redis`, `video-worker` e o container base `nestjs-api` em `tail -f /dev/null`). **Nunca** inicie `npm run start:dev` em background a menos que o usuário peça explicitamente para servir a API HTTP ou durante a verificação manual controlada na Etapa 6 (encerrando-o em seguida).
4. **Convenção de `.env`:** Valores contendo espaços, `<` ou `>` em `nestjs-project/.env` e `nestjs-project/.env.example` devem estar sempre entre aspas duplas (ex.: `MAIL_FROM="StreamTube <noreply@streamtube.com>"`). Corrija `.env.example` caso alguma linha contenha `<` fora de aspas.

### 3.3. Critérios de Prontidão (Readiness) dos Serviços Docker

Após `docker compose up -d`, nunca presuma que um container está pronto apenas por estar `Up`. Verifique explicitamente:

1. **Status dos containers:** `docker compose ps` — todos os 6 serviços (`nestjs-api`, `db`, `mailpit`, `minio`, `redis`, `video-worker`) com status `running` (e `healthy` onde houver healthcheck).
2. **PostgreSQL (`db`):** `docker compose exec -T db pg_isready -U streamtube` → `accepting connections`.
3. **Redis (`redis`):** `docker compose exec -T redis redis-cli ping` → `PONG`.
4. **MinIO (`minio`):** `docker compose exec -T nestjs-api curl -sf http://minio:9000/minio/health/live` → exit code `0`.
5. **FFmpeg no container (`nestjs-api` e `video-worker`):** `docker compose exec -T nestjs-api ffmpeg -version` e `docker compose exec -T video-worker ffprobe -version` → exit code `0`.
6. **Worker ativo (`video-worker`):** `docker compose logs --tail=30 video-worker` → log confirmando bootstrap do `WorkerModule` conectado ao Redis sem erros.

### 3.4. Pirâmide de Testes e Override Obrigatório de Infraestrutura Real

> **ALERTA CRÍTICO CONTRA REPROVA AUTOMÁTICA:**  
> O arquivo legado `.agents/skills/testing-guide-nestjs-project/references/external-systems.md` (escrito na Fase 01, antes da Fase 03 existir) mencionava adaptador em filesystem local para testes de storage. **NÃO USE FILESYSTEM LOCAL PARA SIMULAR O STORAGE!**  
> O documento oficial do desafio (`docs/desafio/desafio.md`, linhas 120, 146, 175, 191 e 243) determina expressamente:
> - *"Sobre o object storage: ele não é uma escolha em aberto. O projeto já aponta para S3 (compatível) — na prática você roda MinIO localmente em Docker (mesma API do S3) e trocaria por S3 em produção."*
> - *"Não mocke o que dá para testar de verdade com a infra do Compose."*
> - *"Reprova automática: Não ter fila, worker e storage reais subindo no Compose."*
> - *"Infra real, testada. Fila, worker e storage precisam subir no Compose e ser exercitados pelos testes: não simule o que dá para rodar de verdade."*

Portanto, a estratégia obrigatória por camada de teste é:

| Sufixo | Camada | Localização | Escopo e Regras Obrigatórias |
| :--- | :--- | :--- | :--- |
| `*.spec.ts` | **Unitário + Compilação de Módulo** | Co-localizado em `nestjs-project/src/**` | Lógica pura de serviços/utilitários com colaboradores mockados via DI (`useValue: { fn: jest.fn() }`) + testes de compilação DI (`*.module.spec.ts`) para todo módulo com imports configurados (`StorageModule`, `VideosModule`, `WorkerModule`). **Proibido** criar testes unitários para Controllers (`*.controller.spec.ts`). |
| `*.integration-spec.ts` | **Integração Real** | Co-localizado em `nestjs-project/src/**` | Exercita **PostgreSQL real (`db`)**, **MinIO real (`http://minio:9000`)**, **Redis/BullMQ real (`redis:6379`)** e **binários `ffprobe`/`ffmpeg` reais** dentro do container. Limpa tabelas, objetos do bucket de teste e filas entre cada teste. |
| `*.e2e-spec.ts` | **End-to-End HTTP + Pipeline Assíncrono** | `nestjs-project/test/videos.e2e-spec.ts` | Sobe `AppModule` real completo com `ValidationPipe` + filtros globais, executa requisições via `supertest`, envia partes reais via `fetch` `PUT` às URLs pré-assinadas do MinIO, conclui o upload, aguarda o processamento real da fila/worker até `status === 'ready'`, valida metadados extraídos, thumbnail gerado, streaming `206 Partial Content` (`Range`), download (`Content-Disposition: attachment`) e fluxo de falha (`status === 'failed'`). |

**Regras técnicas indispensáveis nos testes (descobertas na auditoria do código atual):**

1. **Execução Serial (`--runInBand`):** `npm test` (`jest --runInBand`) e `npm run test:e2e` compartilham a mesma instância PostgreSQL, Redis e MinIO. Jamais remova `--runInBand`.
2. **Fixture de Vídeo Real Gerada via FFmpeg (Zero Binários no Git):** Nos blocos `beforeAll` dos testes de integração de mídia/worker e E2E, gere em `os.tmpdir()` um arquivo `.mp4` H.264 real e determinístico (~15–30 KB, 1.5s, `320x240`, 15 fps) executando `ffmpeg -y -f lavfi -i testsrc=duration=1.5:size=320x240:rate=15 -c:v libx264 -pix_fmt yuv420p -movflags +faststart <tmpPath>` via `execFile`, removendo-o em `afterAll`. Para testar falha permanente de processamento, envie um `Buffer.from('corrupted-non-video-binary-payload')`.
3. **Atualização Obrigatória de `ALL_ENTITIES` (11 arquivos) e `cleanAllTables`:**
   - Pelas regras em `.agents/rules/nestjs-entities.md`, relacionamentos TypeORM devem declarar ambos os lados (`Video.channel` `@ManyToOne` e `Channel.videos` `@OneToMany`).
   - Assim que `Channel` referenciar `Video`, **qualquer** `DataSource` de teste que inclua `Channel` sem incluir `Video` falhará com `Entity metadata for Channel#videos was not found`.
   - Portanto, no mesmo SI que criar `Video` (`SI-03.3`), **atualize `create-test-data-source.ts`** (adicionando `DELETE FROM "videos"` **na primeira linha** de `cleanAllTables()`, antes de `DELETE FROM "channels"`, evitando violação de FK `videos_channel_id_fkey`, e exportando opcionalmente `ALL_ENTITIES` centralizado) e inclua `Video` nos **11 arquivos** existentes:
     1. `nestjs-project/src/users/users.module.spec.ts`
     2. `nestjs-project/src/users/users.service.integration-spec.ts`
     3. `nestjs-project/src/users/entities/user.entity.integration-spec.ts`
     4. `nestjs-project/src/channels/channels.module.spec.ts`
     5. `nestjs-project/src/channels/channels.service.integration-spec.ts`
     6. `nestjs-project/src/channels/entities/channel.entity.integration-spec.ts`
     7. `nestjs-project/src/auth/auth.module.spec.ts`
     8. `nestjs-project/src/auth/auth.service.integration-spec.ts`
     9. `nestjs-project/src/auth/entities/refresh-token.entity.integration-spec.ts`
     10. `nestjs-project/src/auth/entities/verification-token.entity.integration-spec.ts`
     11. `nestjs-project/src/database/migrations.integration-spec.ts` (incluindo também a nova migration `CreateVideos...`, `'videos'` em `MANAGED_TABLES`, `DROP TYPE IF EXISTS "public"."videos_status_enum" CASCADE`, ajuste de `toHaveLength(3)` e restauração completa em `afterAll`).
4. **Isolamento do `ThrottlerGuard` Global (`10 req/min`):**
   - `AuthModule` registra `{ provide: APP_GUARD, useClass: ThrottlerGuard }` globalmente com limite de 10 requisições/minuto por IP.
   - Nos endpoints de leitura contínua de mídia (`stream`, `download`, `thumbnail`), aplique `@SkipThrottle()` para que players HTML5 fazendo múltiplas requisições `Range` nunca recebam `429 Too Many Requests`.
   - Em `test/videos.e2e-spec.ts`, injete `ThrottlerStorage` e chame `throttlerStorage.storage.clear()` no `beforeEach` (espelhando `test/auth.e2e-spec.ts`).
5. **Isolamento entre `video-worker` do Compose e Testes de Integração de Fila:**
   - Como o container `video-worker` estará ativo consumindo a fila padrão `video-processing`, testes de integração que quiserem inspecionar um job parado em estado `waiting` na fila (ex.: teste isolado do publisher) devem usar uma fila prefixada/nomeada de teste via override de `queueConfig` (ou pausar a fila com `await queue.pause()` no `beforeAll` e restaurar `await queue.resume()` no `afterAll`), enquanto o teste E2E (`test/videos.e2e-spec.ts`) usa a fila real para exercitar o consumo ponta a ponta pelo worker!
6. **Fechamento Limpo de Handles (`afterAll`):** Todo teste de módulo, integração ou E2E deve fechar explicitamente conexões (`await app.close()`, `await module.close()`, `await dataSource.destroy()`, `s3Client.destroy()`) em `afterAll` para o Jest encerrar sem open handles.

### 3.5. Definition of Done (DoD) Técnica Obrigatória

Nenhuma etapa de código (e muito menos a entrega final) é considerada concluída sem que **todos** os comandos abaixo passem dentro do container com código de saída `0`:

```powershell
# 1. Migrations aplicam sem erro
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run migration:run

# 2. Suíte completa de testes unitários + integração (serial)
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm test

# 3. Suíte completa de testes E2E
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run test:e2e

# 4. Checagem estrita de tipos TypeScript (0 erros)
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npx tsc --noEmit

# 5. Build completo de produção NestJS
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run build

# 6. ESLint + Prettier (0 erros)
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run lint

# 7. Exportação atualizada do contrato OpenAPI
wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run openapi:export
```

---

## 4. Decisões Arquiteturais da Fase 03 (`TD-01` a `TD-08`) e Especificação Alvo

Após investigação minuciosa da base de código existente (`nestjs-project/`), do diagrama C4 (`docs/diagrams/software-arch.mermaid`), das decisões herdadas (`phase-01-configuracao-base`, `phase-02-auth`, `openapi-docs-nestjs`) e dos requisitos não funcionais de `docs/desafio/desafio.md` e `docs/project-plan.md`, ficam definidas as **8 Decisões Técnicas (`TD-01` a `TD-08`)** que comporão `docs/decisions/technical-decisions-phase-03-videos.md`.

### 4.1. Catálogo das 8 Decisões Técnicas (para `research` e `plan-resolve`)

#### `TD-01`: Tecnologia de Fila de Processamento em Segundo Plano
- **Scope:** `Backend`
- **Capability:** `Serviço de processamento em segundo plano (filas)`
- **Opções a avaliar no `research`:**
  - **Option A: BullMQ + Redis (`@nestjs/bullmq` + `bullmq` sobre `redis:7-alpine`)** — Pacote oficial do ecossistema NestJS 11 (`@nestjs/bullmq@^11.x`), controle nativo de concorrência por worker, retentativas automáticas com backoff exponencial, deduplicação por `jobId`, eventos de falha definitiva (`@OnWorkerEvent('failed')`) e broker Redis extremamente leve (~30 MB RAM) no Docker Compose.
  - **Option B: RabbitMQ (`@nestjs/microservices` RMQ + `amqplib` sobre `rabbitmq:4-management-alpine`)** — Broker AMQP tradicional robusto com exchanges e routing keys, porém exige topologia manual de Dead Letter Exchanges + filas TTL para implementar backoff exponencial e consome ~5x mais memória no Compose.
  - **Option C: Fila baseada em PostgreSQL (`pg-boss`)** — Reaproveita o PostgreSQL existente via `FOR UPDATE SKIP LOCKED`, mas acopla carga de polling ao banco transacional e não materializa um container dedicado de fila no `compose.yaml` (conflitando com o critério explícito de infraestrutura do desafio: *"serviços de object storage, fila e worker subindo via docker compose"*).
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (`@nestjs/bullmq` + `bullmq` com `redis:7-alpine`)**.
- **Libraries:** `@nestjs/bullmq, bullmq`

#### `TD-02`: SDK de Object Storage S3/MinIO, Organização de Bucket/Chaves e Endpoints de Assinatura
- **Scope:** `Backend`
- **Capability:** `Serviço de armazenamento de arquivos (vídeos e thumbnails)`
- **Opções a avaliar no `research`:**
  - **Option A: AWS SDK v3 oficial (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, `forcePathStyle: true`) com bucket único (`streamtube`) particionado por prefixos (`videos/{channelId}/{videoId}/...` e `thumbnails/{channelId}/{videoId}/...`) e suporte a endpoint interno + público de assinatura** — Compatibilidade 100% nativa tanto com MinIO local em Docker (`http://minio:9000`) quanto com AWS S3 em produção sem alterar uma linha de código; inicialização idempotente do bucket (`HeadBucket` + `CreateBucket`) em `OnModuleInit`; separação limpa entre `STORAGE_ENDPOINT` (comunicação container-a-container via host `minio`) e `STORAGE_PUBLIC_ENDPOINT` (opcional, default igual a `STORAGE_ENDPOINT`, permitindo assinar URLs para clientes externos em ambientes com host público distinto sem quebrar a assinatura SigV4 do header `Host`).
  - **Option B: AWS SDK v3 oficial com múltiplos buckets separados (`videos` privado + `thumbnails` público)** — Isola políticas de acesso no nível de bucket, mas duplica configuração de variáveis de ambiente, provisionamento e limpeza em testes sem ganho funcional nesta fase.
  - **Option C: SDK proprietário do MinIO (`minio` npm package)** — API mais simples para MinIO, mas acopla o código a um cliente específico do fornecedor em vez do SDK modular oficial AWS S3 v3 e carece da mesma granularidade de comandos tipados (`UploadPartCommand`, `GetObjectCommand` com `Range`).
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, bucket único com prefixos estruturados e `forcePathStyle: true`)**.
- **Libraries:** `@aws-sdk/client-s3, @aws-sdk/s3-request-presigner`

#### `TD-03`: Estratégia de Upload de Vídeos de até 10GB sem Travar a API e Pré-Cadastro em Rascunho
- **Scope:** `Cross-layer` *(define o protocolo de handshake HTTP entre cliente/futuro frontend, API e Object Storage)*
- **Capability:** `Transversal — covers: "Upload de vídeos com suporte a arquivos de até 10GB sem impacto na performance", "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload"`
- **Opções a avaliar no `research`:**
  - **Option A: S3 Presigned Multipart Upload orquestrado pela API (`init` → `PUT` direto de partes ao S3/MinIO → `complete` / `abort`)** — No início (`POST /videos/uploads`), a API recebe apenas metadados JSON leves (`filename`, `contentType`, `sizeBytes <= 10 GiB`, `title?`), pré-cadastra imediatamente o vídeo no banco com `status = 'draft'` e `short_code` único, inicia um Multipart Upload no S3 (`CreateMultipartUploadCommand`) e devolve `videoId`, `shortCode`, `uploadId`, `partSize` (default `50 MiB`) e as URLs pré-assinadas de cada parte (`UploadPartCommand`), além de permitir re-assinar partes específicas caso a URL expire em conexões lentas. O cliente envia os bytes diretamente ao MinIO/S3 (zero tráfego binário na API NestJS; memória constante O(1) na API; retomada granular por parte em caso de queda de rede) e ao final chama `POST /videos/:id/complete` com `{ parts: [{ partNumber, etag }] }`, momento em que a API finaliza o objeto no S3 (`CompleteMultipartUploadCommand`), valida existência via `HeadObjectCommand`, muda o status para `'processing'` e publica o job na fila.
  - **Option B: Presigned `PUT` Único (`PutObjectCommand`)** — Inviável para 10 GB: o protocolo S3 impõe limite rígido de **5 GiB** por requisição `PUT` simples, além de não permitir retomada parcial se a conexão cair aos 9 GB.
  - **Option C: Servidor intermediário protocolo `tus` (`@tus/server` + `@tus/s3-store`)** — Suporta retomada e 10 GB, mas exige passar todos os 10 GB por um processo Node/container intermediário fazendo proxy para o S3, aumentando consumo de I/O, CPU e superfície de infraestrutura.
  - **Option D: Upload `multipart/form-data` via streaming pela API NestJS (`busboy` / `@aws-sdk/lib-storage`)** — Mantém conexões HTTP da API abertas durante todo o upload de 10 GB, consumindo conexões, workers e banda da API (explicitamente contraindicado no desafio).
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (S3 Presigned Multipart Upload orquestrado pela API com pré-cadastro atômico em `draft`)**.
- **Libraries:** `—` *(usa `@aws-sdk/client-s3` e `@aws-sdk/s3-request-presigner` já fixados em `TD-02`)*

#### `TD-04`: Arquitetura de Execução e Topologia de Container do Video Worker
- **Scope:** `Backend`
- **Capability:** `Serviço de processamento em segundo plano (filas)`
- **Opções a avaliar no `research`:**
  - **Option A: Standalone NestJS Application Context (`NestFactory.createApplicationContext(WorkerModule)`) com entrypoint dedicado (`src/worker.main.ts`) no mesmo pacote `nestjs-project/`, rodando em container `video-worker` separado no `compose.yaml`** — Compartilha entidades TypeORM, `StorageModule`, namespaces `@nestjs/config` e tipagem estrita sem duplicar código nem criar complexidade de monorepo extra, ao mesmo tempo em que isola 100% do consumo de CPU/memória do FFmpeg em um processo/container próprio (`video-worker`, sem servidor HTTP exposto) subindo automaticamente junto com a stack Docker.
  - **Option B: Consumidor BullMQ rodando in-process dentro do próprio container `nestjs-api`** — Compartilha memória e event loop da API HTTP com processos FFmpeg pesados e viola o requisito de container `worker` dedicado subindo no `compose.yaml`.
  - **Option C: Microserviço Node.js separado em novo diretório raiz (`video-worker/`) com `package.json` próprio** — Duplica configurações TypeScript/ESLint, schemas de banco/config, clientes S3 e dificulta manutenção sincronizada da entidade `Video`.
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (`NestFactory.createApplicationContext(WorkerModule)` em `src/worker.main.ts` + serviço `video-worker` dedicado no `compose.yaml`)**.
- **Libraries:** `—`

#### `TD-05`: Extração de Metadados (`ffprobe`) e Geração de Thumbnail (`ffmpeg`) para Arquivos de até 10GB
- **Scope:** `Backend`
- **Capability:** `Transversal — covers: "Processamento automático do vídeo após upload (extração de duração e metadados)", "Geração automática de thumbnail a partir de um frame do vídeo"`
- **Opções a avaliar no `research`:**
  - **Option A: Invocação direta de `ffprobe` e `ffmpeg` via `node:child_process` (`execFile` assíncrono, sem shell) consumindo URL S3 GET pré-assinada de curta duração via HTTP Range** — Zero wrapper npm depreciado, imune a command injection (`execFile` com array de argumentos), e aproveita o suporte nativo a HTTP `Range` do FFmpeg/ffprobe sobre S3/MinIO: mesmo para um vídeo de **10 GB**, o `ffprobe` lê apenas os átomos de cabeçalho (`moov`) e o `ffmpeg -ss <seek> -i <presignedUrl> -frames:v 1` faz seek direto via Range, decodifica um único frame JPEG (`scale='min(1280,iw)':-2`) e grava o thumbnail (~50–150 KB) no MinIO em sub-segundo, **sem baixar 10 GB para disco temporário `/tmp`**. O instante do frame usa cálculo adaptativo (`duration > 1 ? Math.min(Math.floor(duration * 0.1), 5) : 0`) para funcionar tanto em vídeos curtos (< 1s) quanto em vídeos longos.
  - **Option B: Download completo do objeto do S3 para `/tmp` local seguido de `ffmpeg` em arquivo local** — Simples, porém exige ≥ 10 GB de disco efêmero por job concorrente e minutos de transferência de rede desnecessária antes de extrair um único frame.
  - **Option C: Wrapper npm `fluent-ffmpeg`** — Biblioteca oficialmente arquivada/descontinuada em maio de 2025, sem manutenção ativa nem tipagem nativa moderna.
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (`execFile` direto sobre binários `ffprobe`/`ffmpeg` com entrada via Presigned S3 GET URL e seek por HTTP Range)**.
- **Libraries:** `—` *(binários de sistema `ffmpeg`/`ffprobe` instalados via `apt-get` no `Dockerfile.dev` + `node:child_process` nativo)*

#### `TD-06`: Estratégia de Geração de URL Única por Vídeo
- **Scope:** `Cross-layer` *(identificador público consumido em rotas da API e futuras rotas `/watch/:shortCode` do frontend)*
- **Capability:** `URL única por vídeo, sem conflito com outros vídeos`
- **Opções a avaliar no `research`:**
  - **Option A: Código curto aleatório URL-safe de 11 caracteres (`short_code`, alfabeto `[A-Za-z0-9_-]`, estilo YouTube) gerado via CSPRNG nativo (`node:crypto`) com constraint `UNIQUE` no PostgreSQL e retry em colisão (`23505`)** — Espaço de $64^{11} \approx 7,37 \times 10^{19}$ combinações, URL curta e imutável (editar o título na Fase 04 não quebra links compartilhados), não enumerável (essencial para vídeos `unlisted` previstos nas Fases 04/05), zero dependência ESM-only problemática no Jest (`nanoid@5`), e garantia absoluta de unicidade no banco seguindo o mesmo padrão de retry por código PostgreSQL `23505` já estabelecido em `ChannelsService`.
  - **Option B: Slug textual derivado do título + sufixo aleatório (`meu-video-x9k2`)** — No pré-cadastro automático do rascunho o usuário ainda não definiu o título final; além disso, vaza palavras do título em links `unlisted` e fica defasado quando o título é editado na Fase 04.
  - **Option C: Expor diretamente o `id` UUIDv4 na URL pública** — 36 caracteres verbosos; contraria a diretriz explícita de *"URL curta e única"* em `docs/project-plan.md` (§ 4).
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (`short_code` CSPRNG Base64url de 11 caracteres via `node:crypto` + índice `UNIQUE` + retry `23505`)**.
- **Libraries:** `—`

#### `TD-07`: Estratégia de Streaming (`206 Partial Content`) e Download do Vídeo
- **Scope:** `Cross-layer`
- **Capability:** `Transversal — covers: "Reprodução via streaming (sem necessidade de download completo)", "Download do vídeo pelo usuário"`
- **Opções a avaliar no `research`:**
  - **Option A: Endpoints REST de Streaming com suporte a HTTP `Range` (`206 Partial Content`) e Download (`Content-Disposition: attachment`) via pipe não-bufferizado (`Readable`) do S3 `GetObjectCommand`** — `GET /videos/:shortCode/stream` valida `status === 'ready'`, interpreta o header `Range: bytes=start-end` conforme RFC 9110 (limitando o chunk máximo padrão, ex. `5 MiB` quando `end` é aberto `bytes=0-`, ou retornando `200 OK` com `Accept-Ranges: bytes` quando `Range` está ausente e `416 Range Not Satisfiable` quando fora dos limites), solicita apenas a faixa de bytes ao MinIO/S3 (`GetObjectCommand({ Bucket, Key, Range })`) e faz pipe direto do `Readable` stream com memória O(1) (~64 KB) na API. `GET /videos/:shortCode/download` faz stream não-bufferizado com `Content-Disposition: attachment; filename="<sanitized>.<ext>"`, e `GET /videos/:shortCode/thumbnail` entrega o JPEG gerado. Funciona nativamente em `<video>` HTML5, `curl` no host (`localhost:3000`) e `supertest` dentro do container sem problemas de mismatch de assinatura `Host` (`minio:9000` vs `localhost:9000`) e respeita regras de domínio/visibilidade futuras.
  - **Option B: Redirecionamento `302 Found` para Presigned GET URL do S3/MinIO** — Descarrega a transferência de leitura para o storage, mas em ambiente Docker local sem reverse proxy exige expor e sincronizar o `Host` público do MinIO para não falhar com `403 SignatureDoesNotMatch`, além de dificultar asserções diretas de `206 Partial Content` via `supertest` na API.
  - **Option C: Transcodificação multi-bitrate HLS/DASH (`.m3u8` + segmentos `.ts`/`.m4s`)** — Excelente para streaming adaptativo em larga escala, mas multiplica tempo de CPU do worker e armazenamento por vídeo, extrapolando o escopo da Fase 03 (*"extração de duração e metadados"* + *"geração de thumbnail"* + *"reprodução via streaming"* + *"download do vídeo"*).
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (Streaming HTTP Range `206 Partial Content` + Download `attachment` via stream S3 `GetObject` não-bufferizado na API)**.
- **Libraries:** `—`

#### `TD-08`: Ciclo de Status do Vídeo e Política de Falhas e Retentativas no Processamento
- **Scope:** `Backend`
- **Capability:** `Transversal — covers: "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload", "Processamento automático do vídeo após upload (extração de duração e metadados)", "Serviço de processamento em segundo plano (filas)"`
- **Opções a avaliar no `research`:**
  - **Option A: Máquina de estados persistida via PostgreSQL Enum (`draft` → `processing` → `ready` | `failed`) com classificação de erros transitórios vs permanentes no BullMQ** — Quatro estados explícitos no banco: `draft` (pré-cadastrado ao iniciar upload multipart), `processing` (após `CompleteMultipartUpload` bem-sucedido e enfileiramento com `jobId: video.id`), `ready` (metadados + thumbnail persistidos de forma idempotente pelo worker) e `failed` (com coluna `error_reason` preenchida). Erros transitórios (falha de rede/timeout S3) sofrem até `3` tentativas com backoff exponencial (`1000ms` base); erros permanentes (objeto inexistente, arquivo corrompido sem stream de vídeo ou `ffprobe` inválido) lançam `UnrecoverableError` do BullMQ para abortar retentativas inúteis imediatamente. O hook `@OnWorkerEvent('failed')` captura o esgotamento final das tentativas, grava `status = 'failed'` e `error_reason` no PostgreSQL com `try/catch` seguro (sem derrubar o processo do worker, conforme `.agents/rules/nestjs-services.md`).
  - **Option B: Falha imediata na primeira exceção sem retentativas (`attempts: 1`)** — Qualquer oscilação momentânea de rede entre `video-worker` e `minio` marca permanentemente um upload legítimo de 10 GB como `failed`, forçando re-upload pelo usuário.
  - **Option C: Tabela separada de histórico de eventos (`video_processing_events`) sem coluna `status` direta em `videos`** — Complica queries de listagem e leitura rápida de status (`O(1)` na linha de `videos`) exigidas na Fase 03 e no painel do canal da Fase 04.
- **Recomendação / Escolha Alvo (`plan-resolve`):** **Option A (Enum `draft | processing | ready | failed` na tabela `videos` + BullMQ `attempts: 3` exponencial + `UnrecoverableError` para mídia inválida + transição segura em `@OnWorkerEvent('failed')`)**.
- **Libraries:** `—`

---

### 4.2. Matriz de Cobertura das 9 Capabilities (`docs/project-plan.md`)

Esta matriz garante que `plan-validate` encontre **100% das capabilities da Fase 03 cobertas** (zero issues `MD-N` de capability descoberta):

| Capability literal em `docs/project-plan.md` (Fase 03) | Coberta por TD(s) |
| :--- | :--- |
| `Serviço de armazenamento de arquivos (vídeos e thumbnails)` | `phase-03-videos/TD-02` |
| `Serviço de processamento em segundo plano (filas)` | `phase-03-videos/TD-01`, `phase-03-videos/TD-04`, `phase-03-videos/TD-08` |
| `Upload de vídeos com suporte a arquivos de até 10GB sem impacto na performance` | `phase-03-videos/TD-03` |
| `Pré-cadastro automático do vídeo como rascunho ao iniciar o upload` | `phase-03-videos/TD-03`, `phase-03-videos/TD-08` |
| `Processamento automático do vídeo após upload (extração de duração e metadados)` | `phase-03-videos/TD-05`, `phase-03-videos/TD-08` |
| `Geração automática de thumbnail a partir de um frame do vídeo` | `phase-03-videos/TD-05` |
| `URL única por vídeo, sem conflito com outros vídeos` | `phase-03-videos/TD-06` |
| `Reprodução via streaming (sem necessidade de download completo)` | `phase-03-videos/TD-07` |
| `Download do vídeo pelo usuário` | `phase-03-videos/TD-07` |

---

### 4.3. Especificação Técnica Alvo (Referência para `plan-build`)

#### A. Data Model (`videos` + relação inversa em `channels`)

Tabela `videos` (`@Entity('videos')`):

| Coluna | Tipo PostgreSQL | Constraints | Notas |
| :--- | :--- | :--- | :--- |
| `id` | `uuid` | PK, generated (`uuid`) | Identificador interno |
| `channel_id` | `uuid` | FK → `channels.id` (`ON DELETE CASCADE`), `NOT NULL` | Canal dono do vídeo |
| `short_code` | `varchar(16)` | `UNIQUE`, `NOT NULL` | Identificador público de 11 chars `[A-Za-z0-9_-]` (`TD-06`) |
| `title` | `varchar(255)` | `NOT NULL` | Informado no init ou derivado do nome do arquivo (sem extensão) |
| `status` | `enum` (`videos_status_enum`) | `NOT NULL`, default `'draft'` | Valores: `'draft'`, `'processing'`, `'ready'`, `'failed'` (`TD-08`) |
| `original_filename` | `varchar(255)` | `NOT NULL` | Nome original do arquivo enviado |
| `mime_type` | `varchar(100)` | `NOT NULL` | Ex.: `video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska` |
| `size_bytes` | `bigint` | `NOT NULL` | Declarado no init (≤ `10737418240` = 10 GiB) e atualizado pelo tamanho real pós-processamento (nota: TypeORM mapeia `bigint` como `string` no JS — converter/tipar explicitamente!) |
| `storage_key` | `varchar(512)` | `NOT NULL` | Ex.: `videos/{channelId}/{videoId}/source.mp4` |
| `upload_id` | `varchar(512)` | `NULLABLE`, `select: false` | `UploadId` S3 multipart ativo enquanto `status = 'draft'`; limpo ao concluir/abortar |
| `thumbnail_key` | `varchar(512)` | `NULLABLE` | Ex.: `thumbnails/{channelId}/{videoId}/thumbnail.jpg` |
| `duration_seconds` | `integer` | `NULLABLE` | Duração arredondada em segundos extraída pelo `ffprobe` |
| `width` | `integer` | `NULLABLE` | Largura em pixels extraída pelo `ffprobe` |
| `height` | `integer` | `NULLABLE` | Altura em pixels extraída pelo `ffprobe` |
| `video_codec` | `varchar(64)` | `NULLABLE` | Ex.: `h264`, `vp9`, `av1`, `hevc` |
| `error_reason` | `text` | `NULLABLE` | Motivo da falha caso `status = 'failed'` |
| `created_at` | `timestamp` | `NOT NULL`, `@CreateDateColumn()` | |
| `updated_at` | `timestamp` | `NOT NULL`, `@UpdateDateColumn()` | |

- **Relações:** `Video` `@ManyToOne(() => Channel, (channel) => channel.videos, { onDelete: 'CASCADE' })` com `@JoinColumn({ name: 'channel_id' })`; em `Channel`, adicionar `@OneToMany(() => Video, (video) => video.channel) videos: Video[]`.
- **Índices:** `(short_code)` unique, `(channel_id)`, `(status)`.

#### B. API Contracts & Authorization Matrix

| Método e Rota | Auth | Throttle | Status Sucesso | Responsabilidade |
| :--- | :---: | :---: | :---: | :--- |
| `POST /videos/uploads` | JWT Bearer | Padrão | `201 Created` | Valida canal do usuário, MIME e `sizeBytes <= 10 GiB`; cria `Video` em `draft` com `short_code` único; abre Multipart Upload no S3/MinIO; retorna `{ id, shortCode, status, title, uploadId, partSize, totalParts, parts: [{ partNumber, uploadUrl }] }`. |
| `POST /videos/:id/parts/:partNumber/presign` | JWT Bearer (Owner) | `@SkipThrottle()` | `200 OK` | Re-assina URL de upload de uma parte específica (`1..totalParts`) enquanto `status === 'draft'`, suportando retomada após expiração de URL em uploads longos de 10 GB. |
| `POST /videos/:id/complete` | JWT Bearer (Owner) | Padrão | `200 OK` | Valida propriedade do canal e `status === 'draft'`; chama `CompleteMultipartUpload` + `HeadObject` no S3; muda `status` para `'processing'`, limpa `upload_id` e publica job `process-video` na fila BullMQ. |
| `DELETE /videos/:id/upload` | JWT Bearer (Owner) | Padrão | `204 No Content` | Cancela upload em andamento (`status === 'draft'`), chama `AbortMultipartUpload` no S3 para liberar partes órfãs e remove o rascunho incompleto. |
| `GET /videos/:shortCode` | `@Public()` | `@SkipThrottle()` | `200 OK` | Consulta metadados públicos e status atual do vídeo pela URL única (`id`, `shortCode`, `channelId`, `title`, `status`, `durationSeconds`, `width`, `height`, `videoCodec`, `sizeBytes`, `mimeType`, `thumbnailUrl`, `streamUrl`, `downloadUrl`, `createdAt`). |
| `GET /videos/:shortCode/stream` | `@Public()` | `@SkipThrottle()` | `206` / `200` | Reprodução via streaming sem exigir download completo (`Accept-Ranges: bytes`, `Content-Range`, `Content-Length`, `Content-Type`) quando `status === 'ready'`. |
| `GET /videos/:shortCode/download` | `@Public()` | `@SkipThrottle()` | `200 OK` | Download do arquivo de vídeo com `Content-Disposition: attachment; filename="..."` via stream S3 não-bufferizado quando `status === 'ready'`. |
| `GET /videos/:shortCode/thumbnail` | `@Public()` | `@SkipThrottle()` | `200 OK` | Entrega a imagem `image/jpeg` do thumbnail gerado automaticamente quando `status === 'ready'` e `thumbnail_key` presente. |

#### C. Error Catalog (herdando envelope `{ statusCode, error, message }` de `phase-02-auth/TD-07`)

| Error Code (`error`) | HTTP | Trigger |
| :--- | :---: | :--- |
| `CHANNEL_NOT_FOUND` | `404` | Usuário autenticado sem canal associado |
| `VIDEO_NOT_FOUND` | `404` | `id` ou `shortCode` inexistente |
| `FORBIDDEN_VIDEO_ACCESS` | `403` | Usuário autenticado tenta completar/abortar/assinar vídeo pertencente a outro canal |
| `VIDEO_FILE_TOO_LARGE` | `413` | `sizeBytes > 10737418240` (10 GiB) |
| `UNSUPPORTED_VIDEO_FORMAT` | `400` | `contentType` fora da allowlist (`video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`) |
| `INVALID_VIDEO_STATE` | `409` | Tentativa de completar/abortar upload de vídeo que não está em `'draft'` |
| `UPLOAD_INCOMPLETE` | `400` | Falha na validação/conclusão das partes no S3 (`CompleteMultipartUpload` / `HeadObject`) |
| `VIDEO_NOT_READY` | `409` | Tentativa de streaming, download ou thumbnail enquanto `status !== 'ready'` |
| `INVALID_BYTE_RANGE` | `416` | Header `Range` malformado ou fora dos limites `0..sizeBytes-1` |

#### D. Events / Messages (BullMQ)

- **Queue:** `video-processing`
- **Job name:** `process-video` (`jobId: video.id` para idempotência)
- **Payload:** `{ videoId: string }`
- **Producer:** `VideosService.completeUpload()`
- **Consumer:** `VideoProcessor` (`@Processor('video-processing')`) no processo `video-worker`
- **Delivery semantics:** `at-least-once` (`attempts: 3`, `backoff: { type: 'exponential', delay: 1000 }`, `removeOnComplete: 100`, `removeOnFail: false`; `UnrecoverableError` para mídia corrompida/ausente).

#### E. Infraestrutura Docker Compose (`nestjs-project/compose.yaml`)

| Serviço | Imagem / Build | Portas Host | Healthcheck | Papel |
| :--- | :--- | :--- | :--- | :--- |
| `nestjs-api` | `build: Dockerfile.dev` (Node 25.6 + `ffmpeg`) | `3000:3000` | — | Container de desenvolvimento, CLI, testes e API NestJS |
| `db` | `postgres:17` | `5432:5432` | `pg_isready -U streamtube` | Banco de dados relacional |
| `mailpit` | `axllent/mailpit` | `1025:1025`, `8025:8025` | — | Captura SMTP local |
| `minio` | `minio/minio:latest` | `9000:9000`, `9001:9001` | `curl -f http://localhost:9000/minio/health/live` | Object Storage S3-compatível (vídeos e thumbnails) |
| `redis` | `redis:7-alpine` | `6379:6379` | `redis-cli ping` | Broker da fila BullMQ |
| `video-worker` | `build: Dockerfile.dev` (`command: npm run start:worker:dev`) | — (sem porta HTTP) | Processo ativo | Worker assíncrono FFmpeg/ffprobe consumindo `video-processing` |

---

## 5. Governança Git Flow e Mapa Granular de Commits

### 5.1. Regras Inegociáveis de Git Flow (`AGENTS.md` + `docs/desafio/desafio.md`)

1. **PROIBIDO commitar ou fazer merge direto na `main`:** Qualquer commit direto na `main` gera **reprova automática**. A branch `main` permanece intocada em `8459b2f`.
2. **Branch Base de Trabalho Obrigatória (`feature/implementa-desafio-dev`):** A branch atual **`feature/implementa-desafio-dev` É a branch base de trabalho** deste desafio — ela já deriva de `dev`, já recebeu merge da `main` e já consolida as correções de testes (`--runInBand`), a adaptação nativa para o Antigravity e o enunciado em `docs/desafio/desafio.md` (`b35ee9b`). **Não altere diretamente `dev` nem `main`:** a nova branch de desenvolvimento da fase (**`feature/phase-03-videos`**) deve derivar **diretamente** de `feature/implementa-desafio-dev` na Etapa 0 e ser reintegrada em **`feature/implementa-desafio-dev`** via `git merge --no-ff` na Etapa 8 (ficando eventual PR/merge posterior de `feature/implementa-desafio-dev` para `dev` sob controle manual do desenvolvedor).
3. **Commits pequenos, atômicos e significativos:** Cada micro-etapa concluída e validada gera um commit imediato, curto, em português, com prefixo semântico claro e foco no **"porquê"** da mudança. Nunca misture mudanças cosméticas ou fora de escopo no mesmo commit.

### 5.2. Mapa Completo de Commits Previstos

| Ordem | Branch | Momento do Commit | Mensagem Sugerida (focada no "porquê") |
| :---: | :--- | :--- | :--- |
| **C0** | `feature/implementa-desafio-dev` | Antes de ramificar `feature/phase-03-videos` (registro do plano mestre em `docs/desafio/`) | `docs(desafio): adiciona plano mestre de implementacao da Fase 03` *(seguido de `git checkout -b feature/phase-03-videos` diretamente a partir de `feature/implementa-desafio-dev`)* |
| **C1** | `feature/phase-03-videos` | Fim da Etapa 1 (`research`) | `docs(decisions): mapeia opções e trade-offs arquiteturais da Fase 03 de vídeos` |
| **C2** | `feature/phase-03-videos` | Fim da Etapa 2 (`plan-context` + 1º `plan-validate`) | `docs(plan): consolida contexto herdado e validação inicial da Fase 03` |
| **C3** | `feature/phase-03-videos` | Fim da Etapa 3 (`plan-resolve` + `library-refs.md` + `plan-validate` `clean`) | `docs(plan): fixa decisões técnicas, referências Context7 e validação clean da Fase 03` |
| **C4** | `feature/phase-03-videos` | Fim da Etapa 4 (`plan-build` + `plan-test-specs`) | `docs(plan): gera plano executável phase-03-videos com SIs, contratos e specs E2E` |
| **C5** | `feature/phase-03-videos` | Conclusão do `SI-03.1` | `build(infra): provisiona MinIO, Redis e FFmpeg no Compose com configs tipadas` |
| **C6** | `feature/phase-03-videos` | Conclusão do `SI-03.2` | `feat(storage): abstrai operações S3 multipart, presigned URLs e leitura byte-range` |
| **C7** | `feature/phase-03-videos` | Conclusão do `SI-03.3` | `feat(videos): cria entidade Video, vínculo com Channel e migration CreateVideos` |
| **C8** | `feature/phase-03-videos` | Conclusão do `SI-03.4` | `feat(channels): expõe busca de canal por usuário e exceções de domínio de vídeos` |
| **C9** | `feature/phase-03-videos` | Conclusão do `SI-03.5` | `feat(videos): habilita pré-cadastro em rascunho com URL única e início multipart até 10GB` |
| **C10** | `feature/phase-03-videos` | Conclusão do `SI-03.6` | `feat(videos): conclui e aborta upload multipart publicando job na fila BullMQ` |
| **C11** | `feature/phase-03-videos` | Conclusão do `SI-03.7` | `feat(worker): extrai metadados via ffprobe e gera thumbnail JPEG via FFmpeg Range` |
| **C12** | `feature/phase-03-videos` | Conclusão do `SI-03.8` | `feat(worker): processa fila de vídeos com retentativas, ciclo de status e container dedicado` |
| **C13** | `feature/phase-03-videos` | Conclusão do `SI-03.9` | `feat(videos): expõe rotas REST de consulta, streaming 206 Partial Content, download e thumbnail` |
| **C14** | `feature/phase-03-videos` | Conclusão do `SI-03.10` | `chore(openapi): exporta especificação OpenAPI atualizada com endpoints de vídeos` |
| **C15** | `feature/phase-03-videos` | Fim da Etapa 7 (Documentação IA) | `docs(agents): documenta módulo de vídeos, fila BullMQ, worker e storage em AGENTS e CLAUDE` |
| **C16** | `feature/implementa-desafio-dev` | Fim da Etapa 8 (Fechamento na Branch Base) | `merge: integra Fase 03 de upload, processamento assíncrono e streaming de vídeos em feature/implementa-desafio-dev` |

*(Observação: Caso o `plan-build` gere auto-splits pontuais como `SI-03.X.1` / `SI-03.X.2` devido ao limite de 5 ações/testes por SI, cada sub-SI concluído com testes verdes também receberá seu próprio commit atômico seguindo a mesma convenção).*

---

## 6. Roteiro Passo a Passo de Execução Completa (Etapas 0 a 8)

Siga cada etapa **estritamente na ordem abaixo**, sem pular estágios, validando todos os portões de saída antes de avançar.

---

### Etapa 0 — Pré-flight, Higienização Git, Criação da Branch da Fase 03 e Baseline Verde

#### Objetivo
Garantir que o repositório Git na branch base `feature/implementa-desafio-dev`, a ramificação `feature/phase-03-videos`, os servidores MCP do Antigravity e a suíte atual das Fases 01–02 estejam 100% saudáveis antes de iniciar a Fase 03.

#### Instruções Passo a Passo

1. **Verificar disponibilidade do servidor MCP `context7`:**
   - Cheque se as ferramentas MCP do **Context7** (`resolve-library-id` e `query-docs`) estão presentes nas ferramentas ativas da sessão do Antigravity.
   - **Se NÃO estiverem ativas:** instrua o usuário a adicionar/habilitar o plugin oficial **Context7** (`antigravity-plugins-partners/plugin/context7`) pelo Agent Marketplace do Antigravity (ou habilitar em **Additional Options (`...`) > MCP Servers** / `C:\Users\c\.gemini\config\mcp_config.json`) antes de prosseguir para `research` / `plan-resolve`, pois `AGENTS.md` e `docs/desafio/desafio.md` exigem consulta obrigatória via Context7 MCP.
2. **Higienizar estado do Git no WSL:**
   - Confirme que a branch atual é a **branch base de trabalho `feature/implementa-desafio-dev`** (já derivada de `dev` com merge de `main` e com `docs/desafio/desafio.md` registrado em `b35ee9b`) e reverta alterações acidentais de permissão (`filemode`) em `scripts/sync-openapi.sh`:
     ```powershell
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git branch --show-current
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git checkout -- scripts/sync-openapi.sh
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git status -sb
     ```
3. **Commitar o plano mestre na branch base e derivar diretamente `feature/phase-03-videos`:**
   - Em `feature/implementa-desafio-dev`, commite `docs/desafio/plano-implementacao-desafio.md` (**C0** — caso ainda esteja staged/não commitado) e crie a branch `feature/phase-03-videos` **diretamente a partir de `feature/implementa-desafio-dev`** (sem fazer checkout ou merge em `dev` nem em `main`):
     ```powershell
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/desafio/plano-implementacao-desafio.md
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(desafio): adiciona plano mestre de implementacao da Fase 03"
     wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git checkout -b feature/phase-03-videos
     ```
   - Confirme com `git branch --show-current` que a branch ativa é **`feature/phase-03-videos`** (derivada diretamente de `feature/implementa-desafio-dev`) e que `main` permanece intocada em `8459b2f`.
4. **Subir a infraestrutura atual e validar a Baseline das Fases 01 e 02:**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose up -d
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose ps
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T db pg_isready -U streamtube
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm install
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run migration:run
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm test
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run test:e2e
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npx tsc --noEmit
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project/nestjs-project -d Ubuntu -- docker compose exec -T nestjs-api npm run lint
   ```

#### Portão de Saída da Etapa 0
- Branch atual = `feature/phase-03-videos` (derivada diretamente da branch base `feature/implementa-desafio-dev`), `git status` limpo, Context7 MCP operacional e 100% dos testes/tsc/lint das Fases 01–02 passando dentro do container.

---

### Etapa 1 — Research: Documento de Decisões Técnicas (`research`)

#### Objetivo
Executar formalmente a skill `research` para a Fase 03 e produzir `docs/decisions/technical-decisions-phase-03-videos.md` contendo todas as opções, trade-offs e recomendações fundamentadas para `TD-01` a `TD-08`.

#### Instruções Passo a Passo

1. **Ativar a skill `research`:**
   - Leia integralmente `.agents/skills/research/SKILL.md` (incluindo os bytes finais via `ByteOffset` se truncado).
2. **Ler as fontes obrigatórias da skill:**
   - `docs/project-plan.md` (seção `### Fase 03 — Upload e Processamento de Vídeos`, Stack e Pontos de Atenção).
   - `docs/decisions/technical-decisions-phase-01-configuracao-base.md`, `docs/decisions/technical-decisions-phase-02-auth.md` e `docs/decisions/technical-decisions-openapi-docs-nestjs.md` (para respeitar decisões herdadas sem reabri-las).
   - `nestjs-project/package.json` + `nestjs-project/package-lock.json` (versões instaladas: `@nestjs/common@^11.0.1`, `typeorm@^0.3.28`, `joi@^18.1.2`, etc.).
3. **Consultar documentação oficial via Context7 MCP:**
   - Consulte via Context7 (`resolve-library-id` + `query-docs`) a compatibilidade e APIs de:
     - `@nestjs/bullmq` (compatível com NestJS 11) e `bullmq` (`WorkerHost`, `@Processor`, `@OnWorkerEvent`, `UnrecoverableError`, `FlowProducer`/`Queue`).
     - `@aws-sdk/client-s3` e `@aws-sdk/s3-request-presigner` (`S3Client` com `forcePathStyle: true`, `CreateMultipartUploadCommand`, `UploadPartCommand`, `CompleteMultipartUploadCommand`, `AbortMultipartUploadCommand`, `HeadObjectCommand`, `GetObjectCommand` com `Range`, `PutObjectCommand`, `getSignedUrl`).
4. **Escrever `docs/decisions/technical-decisions-phase-03-videos.md`:**
   - **Frontmatter obrigatório:**
     ```yaml
     ---
     scope_type: phase
     related_phases: [3]
     status: pending
     date: <YYYY-MM-DD>
     scope_description: "Backend infrastructure and domain architecture for Phase 03 video upload (up to 10GB), S3/MinIO object storage, async queue/worker processing with FFmpeg, unique URLs, HTTP Range streaming, and video downloads."
     ---
     ```
   - **Bloco `_Subprojects in scope:_` obrigatório (Coverage Check da skill `research`):**
     - `nestjs-project/` — subprojeto primário: recebe `StorageModule`, configuração de fila BullMQ/Redis, `VideosModule`, `WorkerModule` (FFmpeg/ffprobe), migration `CreateVideos` e novos serviços no `compose.yaml`.
     - `next-frontend/` — **sem decisão aberta neste documento:** a interface de gerenciamento e player de vídeo pertence às Fases 04 e 05 conforme `docs/project-plan.md` e `docs/desafio/desafio.md` (TDs `Cross-layer` aqui definem apenas os contratos REST consumíveis futuramente).
   - **Corpo (`TD-01` a `TD-08` + `## Decisions Summary`):**
     - Estruture exatamente as 8 decisões da Seção 4.1 deste plano, com 2 a 4 opções concisas cada, `**Scope:**`, `**Capability:**` citando literalmente os bullets de `docs/project-plan.md`, `**Context:**`, `**Options:**` (`Pros` / `Cons`), `**Recommendation:**` justificada no contexto do projeto e **`**Decision:** _[pending]_`** (respeitando a regra estrita da skill `research`: quem preenche `Decision:` e muda `status: decided` é o estágio `plan-resolve`, nunca `research`!).
5. **Commit Git Flow (C1):**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/decisions/technical-decisions-phase-03-videos.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(decisions): mapeia opcoes e trade-offs arquiteturais da Fase 03 de videos"
   ```

---

### Etapa 2 — Planejamento Parte 1: Consolidação de Contexto e Validação Inicial (`plan-context` + `plan-validate`)

#### Objetivo
Consolidar todas as fontes da Fase 03 em `docs/phases/phase-03-videos/context.md` (via `plan-context`) e gerar o primeiro diagnóstico em `docs/phases/phase-03-videos/validation.md` (via `plan-validate`).

#### Instruções Passo a Passo

1. **Ativar a skill `plan-context`:**
   - Leia `.agents/skills/plan-context/SKILL.md` e `.agents/skills/plan-context/references/context-output-template.md`.
   - Resolva o identificador para a Fase `03`: arquivo `docs/decisions/technical-decisions-phase-03-videos.md`, diretório alvo **`docs/phases/phase-03-videos/`** (aplicando a regra da Seção 2.2 — sem duplicar `phase-03-`!).
   - Verifique o Step 0.5 (UI scope): nenhum bullet da Fase 03 contém palavras de UI (`Tela`, `Página`, `Área`, `Login`, `UI`), portanto `ui_in_scope = false` (omita `## UI Inventory`).
2. **Registrar e disparar em paralelo os 5 subagents leitores no Antigravity:**
   - Leia `.agents/agents/plan-reader.md`, `decisions-reader.md`, `decisions-detail-reader.md`, `decisions-correlator.md` e `phases-reader.md`.
   - Registre os 5 via `define_subagent` (`EnableWriteTools: false`) e invoque-os em paralelo via `invoke_subagent`:
     - `plan-reader` (`NN = 3`)
     - `decisions-reader` (`mode = phase`, `identifier = phase-03-videos`)
     - `decisions-detail-reader` (`mode = phase`, `identifier = phase-03-videos`)
     - `decisions-correlator` (`mode = phase`, `identifier = phase-03-videos`, `scope_prose` = bullets da Fase 03)
     - `phases-reader` (`mode = phase`, `NN = 3`, `slug = phase-03-videos`, `depends_on_slices = []`)
3. **Verificar `Filter Trace` e correlações:**
   - Execute a verificação obrigatória do `Filter Trace` (Step 2 do `plan-context`) comparando os arquivos `docs/decisions/technical-decisions-*.md` listados no disco contra o trace de `decisions-reader` e `decisions-detail-reader`.
   - Para `decisions-correlator`, confirme a inclusão de **`openapi-docs-nestjs`** (pois `VideosController` precisa seguir `openapi-docs-nestjs/TD-01..TD-03`: decoradores `@nestjs/swagger` + `openapi.json` exportado!). Docs exclusivos de `next-frontend-*` não precisam ser incluídos pois `next-frontend/` está diferido/fora de escopo nesta fase.
4. **Extrair requisitos de teste e montar `docs/phases/phase-03-videos/context.md`:**
   - Leia `.agents/skills/testing-guide-nestjs-project/SKILL.md` (§ 3 Feature Implementation Checklist).
   - Capture os timestamps reais (`stat -c '%y'`) de todos os arquivos fonte consumidos em `sources_mtime`.
   - Escreva `docs/phases/phase-03-videos/context.md` com `kind: phase`, `name: phase-03-videos` e todas as seções na ordem canônica obrigatória (`Scope`, `Decisions Index`, `Capability Coverage`, `Decisions Detail`, `Inherited Decisions Detail`, `Inherited Conventions`, `Inherited Deferred Capabilities`, `Non-UI / Deferred Capabilities`, `Testing Requirements`).
   - Nota importante em `## Scope`: declare `**Affected subprojects:** nestjs-project/` e `**Deferred subprojects:** next-frontend/ — interface de vídeo pertence às Fases 04 e 05`.
5. **Ativar e executar a skill `plan-validate` (1ª passagem):**
   - Leia `.agents/skills/plan-validate/SKILL.md`.
   - Valide `docs/phases/phase-03-videos/context.md` aplicando os Checks 1 a 7 (Check 8 é suprimido pois a Fase 03 é monolítica, 1 slice).
   - Como `TD-01` a `TD-08` estão `pending`, `plan-validate` emitirá `OQ-1` a `OQ-8` em `### Unresolved Open Questions` e **zero `MD-N`** (pois todas as 9 capabilities já mapeiam para `TD-01..TD-08`). Caso surja qualquer `AMB-N` ou `MD-N`, corrija antes de finalizar a próxima etapa.
   - Grave `docs/phases/phase-03-videos/validation.md` (`status: dirty`, `issue_count: 8` — ou o total exato de questões abertas).
6. **Commit Git Flow (C2):**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/phases/phase-03-videos/context.md docs/phases/phase-03-videos/validation.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): consolida contexto herdado e validacao inicial da Fase 03"
   ```

---

### Etapa 3 — Planejamento Parte 2: Resolução de Decisões, Context7 (`library-refs.md`) e Validação `clean` (`plan-resolve` + `plan-validate`)

#### Objetivo
Resolver todas as pendências em `validation.md`, consolidar as escolhas em `technical-decisions-phase-03-videos.md` e `context.md`, materializar o cache oficial de bibliotecas em `docs/phases/phase-03-videos/library-refs.md` via Context7 MCP e re-executar `plan-validate` até `validation.md` atingir **`status: clean`**.

#### Instruções Passo a Passo

1. **Ativar a skill `plan-resolve`:**
   - Leia `.agents/skills/plan-resolve/SKILL.md` por completo.
   - Verifique que não há nenhum `MD-N` aberto em `validation.md` (se houver algum `MD-N`, volte ao `research` para adicionar o TD faltante, rode `plan-context` e `plan-validate`).
2. **Coletar/Confirmar as decisões (`TD-01` a `TD-08`):**
   - Apresente as questões abertas em lotes de até 4 por chamada de `ask_question` com a opção recomendada marcada como `(Recommended)` em primeiro lugar (ou, se o usuário autorizou execução contínua autônoma seguindo este plano, aplique diretamente as escolhas recomendadas da Seção 4.1):
     - `TD-01` → **Option A** (`@nestjs/bullmq` + `bullmq` com `redis:7-alpine`) | `**Libraries:** @nestjs/bullmq, bullmq`
     - `TD-02` → **Option A** (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, bucket único com prefixos e `forcePathStyle: true`) | `**Libraries:** @aws-sdk/client-s3, @aws-sdk/s3-request-presigner`
     - `TD-03` → **Option A** (S3 Presigned Multipart Upload orquestrado pela API + pré-cadastro `draft`)
     - `TD-04` → **Option A** (`NestFactory.createApplicationContext(WorkerModule)` + container `video-worker` no Compose)
     - `TD-05` → **Option A** (`execFile` `ffprobe`/`ffmpeg` sobre Presigned GET URL com HTTP Range)
     - `TD-06` → **Option A** (`short_code` CSPRNG Base64url de 11 caracteres via `node:crypto` + `UNIQUE` + retry `23505`)
     - `TD-07` → **Option A** (Streaming HTTP Range `206 Partial Content` + Download `attachment` via S3 byte-range stream)
     - `TD-08` → **Option A** (Enum `draft | processing | ready | failed` + BullMQ `attempts: 3` exponencial + `UnrecoverableError` + `@OnWorkerEvent('failed')`)
3. **Aplicar edições determinísticas (Step 4 do `plan-resolve`):**
   - Atualize `docs/decisions/technical-decisions-phase-03-videos.md`: mude o frontmatter `status: decided`, preencha cada `**Decision:**`, insira `**Libraries:**` em `TD-01` e `TD-02`, e atualize a tabela `## Decisions Summary`.
   - Re-dispare `decisions-reader` e `decisions-detail-reader` pós-edição (validando `Filter Trace` e contagem de 8 TDs decididos), atualize `## Decisions Index` e `## Decisions Detail` em `docs/phases/phase-03-videos/context.md`, e atualize o `sources_mtime` de `technical-decisions-phase-03-videos.md` no frontmatter de `context.md` usando `stat -c '%y'`.
   - Em `docs/phases/phase-03-videos/validation.md`, marque todas as issues resolvidas (`status: resolved`, `resolved_by: phase-03-videos/TD-XX`, `issue_count: 0`) e mova-as para `## Resolved Issues` (mantendo `status: dirty` até que o `plan-validate` confirme `clean`).
4. **Materializar `docs/phases/phase-03-videos/library-refs.md` via Context7 MCP (Step 5 do `plan-resolve`):**
   - Busque via Context7 (`resolve-library-id` + `query-docs` em paralelo) a documentação oficial atualizada para as **4 bibliotecas novas** decididas:
     1. `### @nestjs/bullmq` (`BullModule.forRootAsync`, `BullModule.registerQueue`, `@InjectQueue`, `@Processor`, `WorkerHost`, `@OnWorkerEvent`)
     2. `### bullmq` (`Job`, `Queue`, `UnrecoverableError`, opções de `attempts`, `backoff`, `jobId`, `pause`/`obliterate` para testes)
     3. `### @aws-sdk/client-s3` (`S3Client` com `endpoint`, `region`, `credentials`, `forcePathStyle: true`; `HeadBucketCommand`, `CreateBucketCommand`, `CreateMultipartUploadCommand`, `UploadPartCommand`, `CompleteMultipartUploadCommand`, `AbortMultipartUploadCommand`, `PutObjectCommand`, `HeadObjectCommand`, `GetObjectCommand` com `Range` e consumo de `Body` como `Readable`, `DeleteObjectsCommand`/`ListObjectsV2Command`)
     4. `### @aws-sdk/s3-request-presigner` (`getSignedUrl` para `UploadPartCommand` e `GetObjectCommand` com `expiresIn`)
   - Grave `docs/phases/phase-03-videos/library-refs.md` com frontmatter `libs:` (versão alvo, `context7_id`, `fetched_at`) e `sources_mtime:` real via `stat -c '%y'`.
5. **Re-executar `plan-validate` (2ª passagem — Fechamento Obrigatório em `clean`):**
   - Rode `plan-validate` sobre `context.md` atualizado.
   - Confirme que `docs/phases/phase-03-videos/validation.md` termina com:
     ```yaml
     status: clean
     issue_count: 0
     ```
   - **Bloqueio rígido:** Nunca avance para `plan-build` se `status` em `validation.md` não for literalmente `clean`! (Se qualquer nova inconsistência for apontada, repita `plan-resolve ↔ plan-validate` até fechar em `clean`).
6. **Commit Git Flow (C3):**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/decisions/technical-decisions-phase-03-videos.md docs/phases/phase-03-videos/context.md docs/phases/phase-03-videos/validation.md docs/phases/phase-03-videos/library-refs.md
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): fixa decisoes tecnicas, referencias Context7 e validacao clean da Fase 03"
   ```

---

### Etapa 4 — Planejamento Parte 3: Construção do Plano Executável e Test Specs (`plan-build` + `plan-test-specs`)

#### Objetivo
Gerar `docs/phases/phase-03-videos/phase-03-videos.md` seguindo estritamente o formato de duas fases (`phase-a.md` + `phase-b.md`) da skill `plan-build` e, caso SIs de controller wiring emitam placeholders `**Test Specs:** _pending /plan-test-specs_`, materializar os planos de teste E2E via `plan-test-specs`.

#### Instruções Passo a Passo

1. **Ativar a skill `plan-build` e rodar os Gates 1 a 10:**
   - Leia `.agents/skills/plan-build/SKILL.md` e `.agents/skills/plan-build/phase-a.md`.
   - Leia também `nestjs-project/CLAUDE.md` (exigido em `A1` para parametrizar comandos Docker nos Deliverables) e os 5 templates de Tech Specs aplicáveis em `.agents/skills/plan-build/templates/tech-specs/`:
     1. `data-model.md`
     2. `api-contracts.md`
     3. `auth-matrix.md`
     4. `error-catalog.md`
     5. `events-messages.md` *(obrigatório na Fase 03 por causa da fila!)*
2. **Executar Fase A do `plan-build` (`A1` a `A4.6`):**
   - Escreva o scaffold inicial de `docs/phases/phase-03-videos/phase-03-videos.md` com frontmatter `kind: phase`, `name: phase-03-videos`, `test_specs_aware: true` e `sources_mtime` atualizado via `stat -c '%y'`.
   - Insira incrementalmente (1 `edit_file` por subseção, respeitando a regra *Never write the full file in a single Write call*) as 5 subseções de `## Technical Specifications` na ordem canônica:
     1. `### Data Model` (`Video` + atualização da relação em `Channel`)
     2. `### API Contracts` (os 8 endpoints REST da Seção 4.3.B + `#### Validation Rules`)
     3. `### Authorization Matrix`
     4. `### Error Catalog` (envelope `{ statusCode, error, message }` + códigos da Seção 4.3.C)
     5. `### Events/Messages` (`video.process` / fila `video-processing` conforme template `events-messages.md`)
   - Grave o marcador `<!-- phase-a-complete -->` (`A4.6`).
   - No checkpoint `A5`, prossiga para a Fase B lendo `.agents/skills/plan-build/phase-b.md`.
3. **Executar Fase B do `plan-build` (`B1` a `B7`):**
   - Decomponha o escopo nos **10 Step Implementations (`SI-03.1` a `SI-03.10`)** detalhados na Etapa 5 abaixo, garantindo em `B3` que **nenhum SI exceda `≤ 5 Technical actions`, `≤ 5 Test files` e `≤ 10 Acceptance criteria`**!
   - Valide `B2.5` (todas as 4 libs citadas estão em `library-refs.md`).
   - Escreva um SI por chamada de `edit_file` (`B4`, sem lookback), substitua a sentinela de `## Dependency Map` (`B5`), substitua a sentinela de `## Deliverables` (`B6` — usando os comandos com `docker compose exec nestjs-api ...` conforme `nestjs-project/CLAUDE.md`) e remova a sentinela `<!-- phase-a-complete -->`.
4. **Executar Stage 5 (`plan-test-specs`), se aplicável:**
   - Leia `.agents/skills/plan-test-specs/SKILL.md`.
   - Verifique se algum SI de controller wiring em `phase-03-videos.md` recebeu o placeholder `**Test Specs:** _pending /plan-test-specs_` (ex.: `SI-03.9` com `**Route:**`).
   - Se `PENDING > 0`, execute `plan-test-specs` para gerar `nestjs-project/specs/<feature>.plan.md` (com `subproject: backend`, `runner: jest+supertest`, `target_file: test/videos.e2e-spec.ts`) e atualizar o campo no plano para `**Test Specs:** see \`nestjs-project/specs/<feature>.plan.md\``. Isso garante que o preflight modern de `implement` passe sem abortar por `PENDING TEST SPECS`!
5. **Commit Git Flow (C4):**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add docs/phases/phase-03-videos/ nestjs-project/specs/
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(plan): gera plano executavel phase-03-videos com SIs, contratos e specs E2E"
   ```
   *(Nota: se `nestjs-project/specs/` não existir por ausência de placeholders, faça `git add docs/phases/phase-03-videos/phase-03-videos.md`).*

---

### Etapa 5 — Implementação Guiada Passo a Passo (`implement`: `SI-03.1` a `SI-03.10`)

#### Objetivo
Executar a skill `implement` rigorosamente SI a SI, carregando as skills de boas práticas (`nestjs-best-practices`, `typeorm`, `testing-guide-nestjs-project`) e as seções relevantes de `library-refs.md`, validando os testes no container a cada passo, atualizando `docs/phases/phase-03-videos/progress.md` + `task.md`, e realizando **um commit Git Flow significativo ao final de cada SI verde**.

#### Preparação Obrigatória Antes do Loop (`implement` Preflight)
1. Leia `.agents/skills/implement/SKILL.md`, `.agents/skills/nestjs-best-practices/SKILL.md`, `.agents/skills/typeorm/SKILL.md` e `.agents/skills/testing-guide-nestjs-project/SKILL.md`.
2. Confirme a branch ativa (`feature/phase-03-videos`), indexe `docs/phases/phase-03-videos/phase-03-videos.md` por cabeçalhos, crie a lista completa de SIs em `task.md` (`pending`) e inicialize `docs/phases/phase-03-videos/progress.md` com `**Status:** in_progress` e `**SIs:** 0/10 completed`.

---

#### `SI-03.1` — Infraestrutura Docker (MinIO, Redis, FFmpeg), Dependências e Configurações Tipadas
- **Escopo de Implementação:**
  1. Atualizar `nestjs-project/Dockerfile.dev`: instalar `ffmpeg` junto com `procps` e `curl` (`apt-get update && apt-get install -y --no-install-recommends procps curl ffmpeg && rm -rf /var/lib/apt/lists/*`).
  2. Atualizar `nestjs-project/compose.yaml`: adicionar serviços `minio` (`minio/minio:latest`, comando `server /data --console-address ":9001"`, portas `9000:9000` e `9001:9001`, volume nomeado `minio_data`, healthcheck `/minio/health/live`) e `redis` (`redis:7-alpine`, porta `6379:6379`, healthcheck `redis-cli ping`), adicionando `minio` (`service_healthy`) e `redis` (`service_healthy`) em `depends_on` de `nestjs-api`. *(Nota: o serviço `video-worker` entra em `SI-03.8` assim que `src/worker.main.ts` existir, evitando crash-loop de container antes do entrypoint ser criado!)*
  3. Reconstruir e subir os containers (`docker compose up -d --build`) e instalar dentro de `nestjs-api` as dependências de produção: `@nestjs/bullmq`, `bullmq`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`.
  4. Criar `src/config/storage.config.ts` (`registerAs('storage', ...)` com `endpoint`, `publicEndpoint`, `region`, `accessKeyId`, `secretAccessKey`, `bucket`, `presignExpiresSeconds`, `multipartPartSizeBytes`, `maxUploadSizeBytes`) e `src/config/queue.config.ts` (`registerAs('queue', ...)` com `host`, `port`, `videoQueueName`), registrá-los em `AppModule` (`ConfigModule.forRoot({ load: [...] })`), expandir `src/config/env.validation.ts` (schema Joi) e atualizar `.env` e `.env.example` usando hosts do Compose (`REDIS_HOST=redis`, `STORAGE_ENDPOINT=http://minio:9000`).
- **Testes deste SI:**
  - Atualizar/estender `src/config/env.validation.integration-spec.ts` para cobrir validação obrigatória e defaults de `storage` e `queue` + rodar `test/app.e2e-spec.ts`.
- **Verificação + Commit (C5):**
  - Confirmar `docker compose ps` (`minio` e `redis` `healthy`), `ffmpeg -version` no container, testes do SI verdes, `npx tsc --noEmit` e `npm run lint` verdes, atualizar `progress.md` (`1/10`).
  - Commit: `build(infra): provisiona MinIO, Redis e FFmpeg no Compose com configs tipadas`

---

#### `SI-03.2` — Módulo de Object Storage S3-Compatível (`StorageModule` e `StorageService`)
- **Escopo de Implementação:**
  1. Criar `src/storage/storage.service.ts` encapsulando dois `S3Client` (cliente interno apontando para `storage.endpoint` com `forcePathStyle: true` e cliente presigner apontando para `storage.publicEndpoint ?? storage.endpoint` com `forcePathStyle: true`).
  2. Implementar `onModuleInit()` garantindo criação idempotente do bucket (`HeadBucketCommand` → se 404/NotFound, `CreateBucketCommand`).
  3. Implementar métodos tipados e enxutos:
     - `createMultipartUpload(key, contentType): Promise<string>`
     - `getPresignedUploadPartUrl(key, uploadId, partNumber, expiresInSeconds?): Promise<string>`
     - `completeMultipartUpload(key, uploadId, parts): Promise<void>`
     - `abortMultipartUpload(key, uploadId): Promise<void>`
     - `putObject(key, body, contentType): Promise<void>` (usado pelo worker para salvar o thumbnail)
     - `headObject(key): Promise<{ contentLength: number; contentType?: string }>` (lança exceção de domínio clara se ausente)
     - `getObjectStream(key, range?: { start: number; end: number }): Promise<{ stream: Readable; contentLength: number; contentRange?: string; contentType?: string }>`
     - `getPresignedGetUrl(key, expiresInSeconds?): Promise<string>` (usado pelo worker `ffprobe`/`ffmpeg`)
     - `deleteObject(key): Promise<void>` e `onModuleDestroy()` (`s3Client.destroy()`).
  4. Criar `src/storage/storage.module.ts` exportando `StorageService`.
- **Testes deste SI:**
  - `src/storage/storage.module.spec.ts` (Unit: compilação do módulo DI).
  - `src/storage/storage.service.spec.ts` (Unit: lógica de cálculo de `Range`, seleção de cliente presigner e tratamento de erros).
  - `src/storage/storage.service.integration-spec.ts` (Integration contra o **MinIO real**: ciclo completo de `createMultipartUpload` → `fetch(presignedPartUrl, { method: 'PUT', body })` → `completeMultipartUpload` → `headObject` → `getObjectStream` com `Range` parcial → `abortMultipartUpload` → limpeza no `afterEach`).
- **Verificação + Commit (C6):**
  - Rodar testes do SI, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`2/10`).
  - Commit: `feat(storage): abstrai operacoes S3 multipart, presigned URLs e leitura byte-range`

---

#### `SI-03.3` — Entidade `Video`, Relação com `Channel`, Migration `CreateVideos` e Utilitários de Teste
- **Escopo de Implementação:**
  1. Criar `src/videos/entities/video.entity.ts` (`@Entity('videos')`, enum `VideoStatus`, todas as colunas, índices e `@ManyToOne(() => Channel, (channel) => channel.videos, { onDelete: 'CASCADE' })` conforme Seção 4.3.A).
  2. Atualizar `src/channels/entities/channel.entity.ts` adicionando `@OneToMany(() => Video, (video) => video.channel) videos: Video[]`.
  3. Criar `src/videos/videos.module.ts` inicial registrando `TypeOrmModule.forFeature([Video])` e adicioná-lo em `AppModule` para o TypeORM CLI e `autoLoadEntities` descobrirem `Video`.
  4. Gerar a migration via CLI dentro do container (`docker compose exec -T nestjs-api npm run migration:generate -- src/database/migrations/CreateVideos`), inspecionar o SQL gerado e aplicá-la (`npm run migration:run`).
  5. Atualizar `src/test/create-test-data-source.ts` (`DELETE FROM "videos"` antes de `DELETE FROM "channels"` em `cleanAllTables`) e incluir `Video` nos **11 arquivos** existentes listados na Seção 3.4 (incluindo atualização completa de `src/database/migrations.integration-spec.ts` para validar as 3 migrations e reversão da última).
- **Testes deste SI:**
  - `src/videos/entities/video.entity.integration-spec.ts` (Integration no PostgreSQL real: defaults `status = 'draft'`, timestamps, unicidade de `short_code`, `select: false` em `upload_id`, validação do enum `VideoStatus`, FK `channel_id` e cascade delete ao remover canal).
  - Re-executar `src/database/migrations.integration-spec.ts` e os testes de entidade/serviço existentes para provar zero regressão em `ALL_ENTITIES`!
- **Verificação + Commit (C7):**
  - Rodar toda a suíte unit+integration atual (`npm test`), `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`3/10`).
  - Commit: `feat(videos): cria entidade Video, vinculo com Channel e migration CreateVideos`

---

#### `SI-03.4` — Consulta de Canal por Usuário, Exceções de Domínio de Vídeos e Gerador `short_code`
- **Escopo de Implementação:**
  1. Adicionar `findByUserId(userId: string): Promise<Channel | null>` em `ChannelsService` (`src/channels/channels.service.ts`) — mantendo responsabilidade de `Channel` dentro de `ChannelsModule` (princípio de Single Responsibility de `AGENTS.md`).
  2. Adicionar em `src/common/exceptions/domain.exception.ts` (ou `src/videos/exceptions/video.exceptions.ts` herdando de `DomainException`, de forma que `DomainExceptionFilter` capture automaticamente todas!) as classes do Error Catalog (`ChannelNotFoundException`, `VideoNotFoundException`, `ForbiddenVideoAccessException`, `VideoFileTooLargeException`, `UnsupportedVideoFormatException`, `InvalidVideoStateException`, `UploadIncompleteException`, `VideoNotReadyException`, `InvalidByteRangeException`).
  3. Criar `src/videos/short-code.util.ts` com `generateShortCode(length = 11): string` usando `crypto.randomInt(0, 64)` sobre o alfabeto URL-safe `[A-Za-z0-9_-]` e `src/videos/videos.constants.ts` (`ALLOWED_VIDEO_MIME_TYPES`, `MAX_VIDEO_SIZE_BYTES = 10 * 1024 * 1024 * 1024`, `DEFAULT_PART_SIZE_BYTES = 50 * 1024 * 1024`, `DEFAULT_STREAM_CHUNK_BYTES = 5 * 1024 * 1024`, `VIDEO_PROCESSING_QUEUE = 'video-processing'`, `PROCESS_VIDEO_JOB = 'process-video'`).
- **Testes deste SI:**
  - `src/videos/short-code.util.spec.ts` (Unit: comprimento exato 11, charset `[A-Za-z0-9_-]`, entropia/ausência de colisões em 10.000 amostras).
  - Estender `src/channels/channels.service.spec.ts` e `src/channels/channels.service.integration-spec.ts` para `findByUserId`.
  - Estender `src/common/filters/domain-exception.filter.spec.ts` garantindo mapeamento HTTP exato das novas exceções.
- **Verificação + Commit (C8):**
  - Rodar testes do SI, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`4/10`).
  - Commit: `feat(channels): expoe busca de canal por usuario e excecoes de dominio de videos`

---

#### `SI-03.5` — Pré-Cadastro em Rascunho e Início de Upload Multipart até 10GB (`VideosService`)
- **Escopo de Implementação:**
  1. Criar DTOs de entrada/saída em `src/videos/dto/`: `initiate-upload.dto.ts` (`filename`, `contentType`, `sizeBytes`, `title?` com `class-validator` — sem `@ApiProperty` redundante, apenas JSDoc para o CLI plugin do Swagger!).
  2. Criar `src/videos/videos.service.ts` implementando:
     - `initiateUpload(userId: string, dto: InitiateUploadDto)`: valida MIME permitido (`400 UNSUPPORTED_VIDEO_FORMAT`), valida `sizeBytes > 0` e `sizeBytes <= maxUploadSizeBytes` (`413 VIDEO_FILE_TOO_LARGE` antes de tocar banco ou S3!), busca o canal do usuário (`404 CHANNEL_NOT_FOUND`), calcula `partSize` e `totalParts = Math.ceil(sizeBytes / partSize)` (ex.: para arquivo de 10 GiB com partes de 50 MiB → 205 partes), inicia o Multipart Upload no S3 (`storageService.createMultipartUpload`), persiste `Video` com `status = VideoStatus.DRAFT`, `short_code` único (com loop de retry em erro PostgreSQL `23505` em `short_code`; se falhar ao salvar no banco, compensa chamando `abortMultipartUpload` no S3!) e gera as URLs pré-assinadas das partes (`parts`), ou sob demanda via `presignPart(userId, videoId, partNumber)`.
- **Testes deste SI:**
  - `src/videos/videos.service.spec.ts` (Unit: cálculo de partes até 10 GiB, rejeição `413` acima de 10 GiB, rejeição MIME inválido, derivação de título do filename, retry em colisão de `short_code`, compensação `abortMultipartUpload` se o banco falhar, verificação de dono em `presignPart`).
  - `src/videos/videos.service.integration-spec.ts` (Integration com PostgreSQL + MinIO reais: cria vídeo como `draft` ligado ao `channel_id` correto, gera `short_code` único e retorna URLs pré-assinadas funcionais).
- **Verificação + Commit (C9):**
  - Rodar testes do SI, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`5/10`).
  - Commit: `feat(videos): habilita pre-cadastro em rascunho com URL unica e inicio multipart ate 10GB`

---

#### `SI-03.6` — Conclusão e Cancelamento de Upload Multipart + Publicação na Fila BullMQ
- **Escopo de Implementação:**
  1. Configurar `BullModule.forRootAsync` (usando `queueConfig`) em `AppModule` e `BullModule.registerQueueAsync({ name: VIDEO_PROCESSING_QUEUE })` em `VideosModule` (`src/videos/videos.module.ts`), importando `StorageModule` e `ChannelsModule`.
  2. Criar `src/videos/dto/complete-upload.dto.ts` (`parts: CompleteUploadPartDto[]` com `@ValidateNested({ each: true })`, `@ArrayMinSize(1)`, `partNumber` `@IsInt() @Min(1)`, `etag` `@IsString() @IsNotEmpty()`).
  3. Implementar em `VideosService`:
     - `completeUpload(userId: string, videoId: string, dto: CompleteUploadDto)`: carrega `Video` (incluindo `upload_id` via `addSelect`), valida propriedade do canal (`403 FORBIDDEN_VIDEO_ACCESS`) e `status === DRAFT` (`409 INVALID_VIDEO_STATE`), executa `storageService.completeMultipartUpload` + `storageService.headObject` (convertendo falhas do S3 em `400 UPLOAD_INCOMPLETE`), atualiza `status = VideoStatus.PROCESSING`, `size_bytes` real e `upload_id = null`, e publica na fila BullMQ `videoQueue.add(PROCESS_VIDEO_JOB, { videoId: video.id }, { jobId: video.id, attempts: 3, backoff: { type: 'exponential', delay: 1000 }, removeOnComplete: 100, removeOnFail: false })`.
     - `abortUpload(userId: string, videoId: string)`: valida dono + `status === DRAFT`, chama `storageService.abortMultipartUpload` e remove o registro rascunho.
- **Testes deste SI:**
  - `src/videos/videos.module.spec.ts` (Unit: compilação completa do `VideosModule` com TypeORM, BullMQ, `StorageModule` e `ChannelsModule`).
  - Estender `src/videos/videos.service.spec.ts` (Unit: branches de `completeUpload` e `abortUpload`, autorização, estado inválido, falha de S3).
  - Estender `src/videos/videos.service.integration-spec.ts` (Integration com PostgreSQL + MinIO + Redis reais: enviar bytes reais ao MinIO via URL pré-assinada, chamar `completeUpload`, verificar objeto montado no MinIO, `status === 'processing'` no PostgreSQL e job `process-video` presente na fila Redis com `{ videoId }`).
- **Verificação + Commit (C10):**
  - Rodar testes do SI, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`6/10`).
  - Commit: `feat(videos): conclui e aborta upload multipart publicando job na fila BullMQ`

---

#### `SI-03.7` — Serviço de Mídia FFmpeg/ffprobe (`MediaProbeService`: Metadados + Thumbnail via HTTP Range)
- **Escopo de Implementação:**
  1. Criar `src/videos/media-probe.service.ts` usando `execFile` promisificado (`node:child_process`):
     - `extractMetadata(inputUrl: string): Promise<VideoMediaMetadata>`: executa `ffprobe -v error -print_format json -show_format -show_streams <inputUrl>` (com timeout defensivo, ex.: `60_000ms`), faz parse tipado do JSON, localiza a stream `codec_type === 'video'` (se não houver ou `ffprobe` falhar com erro de formato/corrupção, lança erro tipado `InvalidMediaFileError`), e retorna `{ durationSeconds, width, height, videoCodec }`.
     - `extractThumbnailJpeg(inputUrl: string, durationSeconds: number): Promise<Buffer>`: calcula `seekSeconds = durationSeconds > 1 ? Math.min(Math.floor(durationSeconds * 0.1), 5) : 0`, executa `ffmpeg -hide_banner -loglevel error -ss <seekSeconds> -i <inputUrl> -frames:v 1 -vf "scale='min(1280,iw)':-2" -f image2 -vcodec mjpeg pipe:1` (`encoding: 'buffer'`, `maxBuffer: 10 * 1024 * 1024`), verifica buffer JPEG não vazio (assinatura `0xFF 0xD8`) e o retorna para upload direto ao MinIO.
- **Testes deste SI:**
  - `src/videos/media-probe.service.spec.ts` (Unit: cálculo adaptativo de `seekSeconds`, parsing de JSON do `ffprobe`, rejeição quando não há stream de vídeo, classificação entre erro de formato inválido vs erro transitório de I/O).
  - `src/videos/media-probe.service.integration-spec.ts` (Integration com **MinIO + `ffprobe` + `ffmpeg` reais**: gera fixture `.mp4` `320x240` real em `beforeAll`, sobe no MinIO, gera Presigned GET URL, extrai metadados exatos `width=320`, `height=240`, `videoCodec='h264'`, `durationSeconds >= 1`, gera thumbnail JPEG válido `0xFF 0xD8`, e testa arquivo corrompido lançando `InvalidMediaFileError`).
- **Verificação + Commit (C11):**
  - Rodar testes do SI, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`7/10`).
  - Commit: `feat(worker): extrai metadados via ffprobe e gera thumbnail JPEG via FFmpeg Range`

---

#### `SI-03.8` — Processador BullMQ (`VideoProcessor`), `WorkerModule`, Entrypoint `worker.main.ts` e Serviço `video-worker` no Compose
- **Escopo de Implementação:**
  1. Criar `src/videos/video.processor.ts` (`@Processor(VIDEO_PROCESSING_QUEUE)` estendendo `WorkerHost`):
     - `process(job: Job<{ videoId: string }>)`: busca `Video` no banco; se não existir ou já estiver `READY`, retorna de forma idempotente; verifica objeto via `storageService.headObject` (se 404 → `throw new UnrecoverableError('Source video object missing in storage')`); obtém Presigned GET URL interna (`http://minio:9000/...`); chama `mediaProbeService.extractMetadata` e `extractThumbnailJpeg` (se `InvalidMediaFileError` → `throw new UnrecoverableError(err.message)`); faz upload do JPEG em `thumbnails/${video.channel_id}/${video.id}/thumbnail.jpg` (`image/jpeg`); atualiza `Video` com `status = VideoStatus.READY`, `duration_seconds`, `width`, `height`, `video_codec`, `thumbnail_key`, `size_bytes` real e `error_reason = null`.
     - `@OnWorkerEvent('failed') async onFailed(job: Job<{ videoId: string }> | undefined, error: Error)`: envolto em `try/catch` com `Logger` (nunca relança para não derrubar o processo worker!), verifica se o job esgotou todas as tentativas (`job.attemptsMade >= (job.opts.attempts ?? 1)` ou `error instanceof UnrecoverableError`) e atualiza `Video` no banco para `status = VideoStatus.FAILED` e `error_reason = error.message`.
  2. Criar `src/worker/worker.module.ts` (`ConfigModule.forRoot`, `TypeOrmModule.forRootAsync`, `BullModule.forRootAsync`, `BullModule.registerQueueAsync`, `TypeOrmModule.forFeature([Video, Channel, User])`, `StorageModule`, `MediaProbeService`, `VideoProcessor`) e `src/worker.main.ts` (`NestFactory.createApplicationContext(WorkerModule)` + `app.enableShutdownHooks()` para encerramento gracioso em `SIGTERM`/`SIGINT`).
  3. Adicionar scripts em `nestjs-project/package.json`:
     - `"start:worker": "node dist/worker.main"`
     - `"start:worker:dev": "ts-node -r tsconfig-paths/register src/worker.main.ts"`
  4. Adicionar o serviço **`video-worker`** em `nestjs-project/compose.yaml` (mesmo build `Dockerfile.dev`, volume `.:/home/node/app`, `command: npm run start:worker:dev`, `restart: unless-stopped`, `depends_on` de `db`, `redis` e `minio` com `service_healthy`) e subir com `docker compose up -d`!
- **Testes deste SI:**
  - `src/worker/worker.module.spec.ts` (Unit: compilação completa do `WorkerModule`).
  - `src/videos/video.processor.spec.ts` (Unit: fluxo feliz `ready`, idempotência, `UnrecoverableError` em mídia corrompida, transição `failed` apenas no esgotamento de tentativas em `onFailed`).
  - `src/videos/video.processor.integration-spec.ts` (Integration com PostgreSQL + MinIO + FFmpeg reais: processa vídeo `.mp4` real gravando `status = 'ready'` + thumbnail verificável no MinIO, e processa arquivo inválido gravando `status = 'failed'` + `error_reason`).
- **Verificação + Commit (C12):**
  - Confirmar `docker compose ps` mostrando **`video-worker` `running`** e logs limpos em `docker compose logs video-worker`, rodar testes do SI, `npx tsc --noEmit`, `npm run build` (confirmando geração de `dist/worker.main.js`), `npm run lint`, atualizar `progress.md` (`8/10`).
  - Commit: `feat(worker): processa fila de videos com retentativas, ciclo de status e container dedicado`

---

#### `SI-03.9` — Consulta por URL Única, Streaming HTTP `Range` (`206 Partial Content`), Download, Thumbnail e `VideosController`
- **Escopo de Implementação:**
  1. Implementar métodos de leitura em `VideosService`:
     - `findByShortCode(shortCode: string)`: retorna DTO de resposta enriquecido com metadados, status e caminhos relativos/URLs de `stream`, `download` e `thumbnail` (ou lança `404 VIDEO_NOT_FOUND`).
     - `getVideoStream(shortCode: string, rangeHeader?: string)`: valida `status === VideoStatus.READY` (`409 VIDEO_NOT_READY`); se `rangeHeader` presente, faz parse estrito de `bytes=start-end` (suportando `bytes=start-` limitado por chunk de `5 MiB` e `bytes=-suffix`; qualquer faixa inválida ou `start >= size` lança `416 INVALID_BYTE_RANGE`), chama `storageService.getObjectStream(video.storage_key, { start, end })` e retorna metadados de `206 Partial Content` (`Content-Range`, `Accept-Ranges: bytes`, `Content-Length`, `Content-Type`) + `Readable`; se `rangeHeader` ausente, retorna `200 OK` com `Accept-Ranges: bytes`, `Content-Length: size` e `Readable` completo sem carregar em memória.
     - `getVideoDownload(shortCode: string)`: valida `status === VideoStatus.READY`, sanitiza nome de arquivo e retorna stream S3 com `Content-Disposition: attachment; filename="..."`, `Content-Type` e `Content-Length`.
     - `getThumbnailStream(shortCode: string)`: valida `status === VideoStatus.READY` e `thumbnail_key` presente, retornando stream `image/jpeg`.
  2. Criar `src/videos/videos.controller.ts` fino (sem lógica de negócio, sem `try/catch`, delegando erros aos exception filters conforme `.agents/rules/nestjs-controllers.md` e `.agents/rules/nestjs-layer-separation.md`), com `@Public()` e `@SkipThrottle()` nas rotas públicas de leitura/streaming/download/thumbnail, `@ApiBearerAuth('access-token')` nas rotas autenticadas e documentação `@nestjs/swagger` completa (`@ApiTags('videos')`, `@ApiOperation`, `@ApiResponse` de sucesso + erros usando `getSchemaPath(ApiErrorEnvelope)`), retornando `StreamableFile` nas rotas binárias.
- **Testes deste SI:**
  - Estender `src/videos/videos.service.spec.ts` (Unit: parsing de headers `Range` — `bytes=0-1023`, `bytes=0-`, `bytes=-500`, faixas fora do limite `416`, bloqueio `409 VIDEO_NOT_READY` quando `draft`/`processing`/`failed`).
  - Criar `test/videos.e2e-spec.ts` (E2E completo com PostgreSQL + MinIO + Redis + `video-worker` reais):
    1. Rejeição `401` sem JWT em `POST /videos/uploads`, `POST /videos/:id/complete` e `DELETE /videos/:id/upload`.
    2. Validação `400 VALIDATION_ERROR` em payload inválido e `400 UNSUPPORTED_VIDEO_FORMAT` em MIME não suportado.
    3. Validação `413 VIDEO_FILE_TOO_LARGE` quando `sizeBytes > 10 GiB` e sucesso `201` declarando `sizeBytes = 10_737_418_240` (comprovando cálculo de partes de 10 GiB em milissegundos sem travar a API!).
    4. Pré-cadastro como `draft` com `shortCode` único sem conflito entre múltiplos vídeos e re-assinatura de parte (`POST /videos/:id/parts/1/presign`).
    5. Isolamento de propriedade (`403 FORBIDDEN_VIDEO_ACCESS` quando outro usuário tenta completar/abortar o vídeo).
    6. Cancelamento de upload (`DELETE /videos/:id/upload` retorna `204` e aborta no MinIO).
    7. **Fluxo Feliz Assíncrono Ponta a Ponta Real:** `POST /videos/uploads` → `PUT` dos bytes do `.mp4` real na URL pré-assinada do MinIO → `POST /videos/:id/complete` (`status: 'processing'`) → polling curto em `GET /videos/:shortCode` até o container `video-worker` concluir o job (`status: 'ready'`, `durationSeconds > 0`, `width: 320`, `height: 240`, `videoCodec: 'h264'`) → `GET /videos/:shortCode/thumbnail` retorna `200` `image/jpeg` válido → `GET /videos/:shortCode/stream` com `Range: bytes=0-511` retorna **`206 Partial Content`** com `Content-Range: bytes 0-511/<total>`, `Accept-Ranges: bytes` e exatamente `512` bytes (acesso anônimo sem JWT!) → `GET /videos/:shortCode/stream` com `Range: bytes=99999999-` retorna `416` → `GET /videos/:shortCode/download` retorna `200` com `Content-Disposition: attachment` e bytes idênticos ao MP4 enviado!
    8. **Fluxo de Erro Ponta a Ponta Real:** upload de arquivo corrompido → `complete` → `video-worker` marca `status: 'failed'` no banco → tentativa de `stream` ou `download` retorna `409 VIDEO_NOT_READY`.
- **Verificação + Commit (C13):**
  - Rodar `npm test` + `npm run test:e2e`, `npx tsc --noEmit`, `npm run lint`, atualizar `progress.md` (`9/10`).
  - Commit: `feat(videos): expoe rotas REST de consulta, streaming 206 Partial Content, download e thumbnail`

---

#### `SI-03.10` — Sincronização e Verificação do Contrato OpenAPI (`openapi.json`)
- **Escopo de Implementação:**
  1. Executar `docker compose exec -T nestjs-api npm run openapi:export` para regenerar `nestjs-project/openapi.json` com todos os schemas e rotas do módulo `videos`.
  2. Estender `src/openapi-export.integration-spec.ts` para asserir que:
     - Todas as rotas `/videos/...` possuem `summary` não vazio.
     - As rotas protegidas de `videos` exigem o security scheme `access-token` e as rotas `@Public()` de `videos` não exigem `access-token`.
     - As respostas de erro das rotas de `videos` referenciam `#/components/schemas/ApiErrorEnvelope`.
- **Testes deste SI:**
  - Rodar `src/openapi-export.integration-spec.ts` e `test/swagger.e2e-spec.ts`.
- **Verificação + Fechamento de `progress.md` + Commit (C14):**
  - Rodar suíte completa (`npm test` + `npm run test:e2e`), `npx tsc --noEmit`, `npm run build`, `npm run lint`.
  - Atualizar `docs/phases/phase-03-videos/progress.md` para `**Status:** completed` e `**SIs:** 10/10 completed` (e marcar `task.md` 100% concluído).
  - Commit: `chore(openapi): exporta especificacao OpenAPI atualizada com endpoints de videos`

---

### Etapa 6 — Verificação Ponta a Ponta em Runtime Real (Docker Compose Completo)

#### Objetivo
Validar empiricamente a aplicação rodando por completo via HTTP real, inspecionando containers, PostgreSQL, MinIO e logs do `video-worker`.

#### Instruções Passo a Passo

1. Confirmar que todos os 6 containers (`nestjs-api`, `db`, `mailpit`, `minio`, `redis`, `video-worker`) estão `running`/`healthy` em `docker compose ps`.
2. Iniciar temporariamente a API compilada dentro do container (`npm run build`), executar um teste de fumaça completo via script/comandos HTTP contra `http://localhost:3000` (cadastro + confirmação + login → `POST /videos/uploads` → `PUT` para MinIO → `POST /videos/:id/complete` → aguardar log de sucesso no `video-worker` → `curl -i -H "Range: bytes=0-1023" http://localhost:3000/videos/<shortCode>/stream` validando `HTTP/1.1 206 Partial Content` → `curl -I http://localhost:3000/videos/<shortCode>/download` validando `Content-Disposition: attachment`), e encerrar o processo HTTP temporário ao final (respeitando a regra de não deixar `start:dev` pendurado).

---

### Etapa 7 — Atualização e Paridade da Documentação de IA

#### Objetivo
Cumprir o Requisito 4 de `docs/desafio/desafio.md` com **100% de fidelidade ao código implementado** e **paridade dupla** entre a convenção nativa do Antigravity (`AGENTS.md` + `.agents/rules/`) e os arquivos originais `CLAUDE.md` presentes no repositório (evitando qualquer risco da cláusula de reprova automática *"CLAUDE.md/equivalente inconsistente com o código"*).

#### Instruções Passo a Passo

1. **Atualizar `AGENTS.md` (raiz) e `CLAUDE.md` (raiz) em paridade:**
   - Em `## Architecture (C4 Container Diagram)`: substituir `Message Queue (TBD)` por `Message Queue (Redis / BullMQ) → video processing job queue (video-processing)`.
   - Em `## Repository Structure`: atualizar o status real dos diretórios (`nestjs-project/` com módulos `auth`, `users`, `channels`, `storage`, `videos`, `worker`, `mail`, `swagger`; `next-frontend/` inicializado nas Fases 01–02).
   - Adicionar seção **`## Videos Module (Phase 03)`** documentando com exatidão:
     - Fluxo de upload direto de até 10 GiB via S3 Presigned Multipart Upload (pré-cadastro em `draft` → upload de partes ao MinIO/S3 → `complete` → fila `video-processing`).
     - Ciclo de status (`draft → processing → ready | failed`) e política de retentativas do `video-worker` (`ffprobe` + `ffmpeg` via HTTP Range).
     - Lista exata dos endpoints implementados em `VideosController`, geração de `short_code` de 11 caracteres, streaming `206 Partial Content` e download.
2. **Atualizar `.agents/rules/nestjs-project.md`, `nestjs-project/CLAUDE.md` e criar/sincronizar `nestjs-project/AGENTS.md`:**
   - Documentar os novos serviços do `compose.yaml` (`minio` portas `9000`/`9001`, `redis` porta `6379`, `video-worker` sem porta HTTP), comandos de readiness (`redis-cli ping`, healthcheck do MinIO, `ffmpeg -version`), scripts `npm run start:worker:dev` e `npm run start:worker`, e notas de isolamento de testes com MinIO/Redis/`ThrottlerStorage`.
3. **Atualizar `docs/diagrams/software-arch.mermaid`:**
   - Alterar `ContainerQueue(queue, "Message Queue", "TBD", "Enqueues video processing jobs")` para `ContainerQueue(queue, "Message Queue", "Redis / BullMQ", "Enqueues video processing jobs")`.
4. **Auditoria Anti-Alucinação Obrigatória:**
   - Para **cada** caminho de arquivo, nome de script `npm`, variável de ambiente, serviço do `compose.yaml` e rota HTTP citada nos arquivos de documentação atualizados, verifique programaticamente no disco/código que o item existe com aquele exato nome. *"Documentação que cite arquivos ou comportamentos inexistentes reprova."*
5. **Commit Git Flow (C15):**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git add AGENTS.md CLAUDE.md nestjs-project/AGENTS.md nestjs-project/CLAUDE.md .agents/rules/nestjs-project.md docs/diagrams/software-arch.mermaid
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git commit -m "docs(agents): documenta modulo de videos, fila BullMQ, worker e storage em AGENTS e CLAUDE"
   ```

---

### Etapa 8 — Auditoria Final de Definition of Done, Anti-Reprova e Merge na Branch Base (`feature/implementa-desafio-dev`)

#### Objetivo
Executar a bateria final de verificação técnica, validar todos os Critérios de Aceite item a item e concluir o fluxo Git Flow integrando `feature/phase-03-videos` de volta na branch base de trabalho `feature/implementa-desafio-dev` (sem tocar na `main` nem alterar `dev` diretamente).

#### Instruções Passo a Passo

1. **Executar a bateria completa da Definition of Done (Seção 3.5):**
   - `docker compose ps` (todos os 6 serviços ativos e saudáveis)
   - `docker compose exec -T nestjs-api npm run migration:run` (exit 0)
   - `docker compose exec -T nestjs-api npm test` (100% verde, `--runInBand`)
   - `docker compose exec -T nestjs-api npm run test:e2e` (100% verde)
   - `docker compose exec -T nestjs-api npx tsc --noEmit` (exit 0, zero erros)
   - `docker compose exec -T nestjs-api npm run build` (exit 0)
   - `docker compose exec -T nestjs-api npm run lint` (exit 0, zero erros)
   - Verificar `git status` limpo (nenhum arquivo não rastreado ou modificado pendente, inclusive `openapi.json`).
2. **Conferir integralmente a Seção 7 abaixo (Checklist de Aceite e Anti-Reprova).**
3. **Integrar `feature/phase-03-videos` na branch base `feature/implementa-desafio-dev` via Git Flow (`--no-ff`) — Commit C16:**
   ```powershell
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git checkout feature/implementa-desafio-dev
   wsl --cd /home/c/FullCycle/desafios/mba-ia-greenfield-project -d Ubuntu -- git merge --no-ff feature/phase-03-videos -m "merge: integra Fase 03 de upload, processamento assincrono e streaming de videos em feature/implementa-desafio-dev"
   ```
4. **Verificação final de segurança de branches:**
   - Executar `git log --oneline -n 3 main` para comprovar que `main` permanece **exatamente** no commit original `8459b2f` (zero commits diretos na `main`).
   - Executar `git log --oneline --graph -n 20 feature/implementa-desafio-dev` para comprovar o histórico limpo, atômico e rastreável de Git Flow entre `feature/implementa-desafio-dev` (branch base derivada de `dev` com merge de `main`) e `feature/phase-03-videos`. *(A consolidação final `feature/implementa-desafio-dev → dev` via PR/merge fica sob controle exclusivo do desenvolvedor).*

---

## 7. Checklist Final de Aceite e Blindagem contra Reprova Automática

Antes de declarar o desafio concluído, o agente executor **DEVE** marcar e comprovar com evidência real cada item abaixo:

### 7.1. Critérios de Aceite (`docs/desafio/desafio.md`)

- [ ] **Decisões técnicas:** `docs/decisions/technical-decisions-phase-03-videos.md` existe com `status: decided` e todas as decisões justificadas (fila, estratégia de upload de 10GB, storage/buckets, worker/processamento/thumbnail, URL única, streaming/download e ciclo de status/falhas).
- [ ] **Pasta e artefatos da fase:** `docs/phases/phase-03-videos/` contém `context.md`, `validation.md` (**`status: clean`**), `library-refs.md` (gerado via Context7), `phase-03-videos.md` e `progress.md` (`Status: completed`).
- [ ] **Formato do plano:** `phase-03-videos.md` contém SIs `SI-03.x`, `Technical Specifications` completas (`Data Model`, `API Contracts`, `Authorization Matrix`, `Error Catalog` e **`Events/Messages`**), `Dependency Map` e `Deliverables`.
- [ ] **Upload de até 10GB sem travar + rascunho:** Upload multipart direto ao MinIO/S3 via URLs pré-assinadas funcional para arquivos declarados de até 10 GiB sem trafegar binário pela API, criando automaticamente o vídeo em `status = 'draft'` ao iniciar.
- [ ] **Processamento automático e thumbnail:** Worker consome job automaticamente após `complete`, extrai duração/resolução/codec via `ffprobe` e gera thumbnail JPEG via `ffmpeg` no MinIO.
- [ ] **URL única:** Cada vídeo recebe `short_code` único sem conflito, protegido por índice `UNIQUE` e retry de colisão.
- [ ] **Streaming e download:** `GET /videos/:shortCode/stream` responde `206 Partial Content` a requisições `Range` sem exigir download completo, e `GET /videos/:shortCode/download` disponibiliza o download do arquivo.
- [ ] **Ciclo de status:** Transições `draft → processing → ready` e `draft → processing → failed` refletidas no PostgreSQL e cobertas por testes reais.
- [ ] **Infraestrutura Docker:** MinIO (`minio`), Redis (`redis`) e Worker (`video-worker`) sobem via `docker compose` junto com o backend (`nestjs-api`, `db`, `mailpit`).
- [ ] **Migration e relação:** Migration versionada em `nestjs-project/src/database/migrations/*-CreateVideos.ts` cria a tabela `videos` ligada a `channels`.
- [ ] **Definition of Done completa:** `npm test` verde + `npm run test:e2e` verde + `npx tsc --noEmit` (código 0) + `npm run lint` verde (tudo executado dentro do container).
- [ ] **Git Flow:** Trabalho conduzido em `feature/phase-03-videos` derivada diretamente da branch base `feature/implementa-desafio-dev` (que já deriva de `dev` com merge de `main`) e reintegrada em `feature/implementa-desafio-dev` via `--no-ff`, sem nenhum commit direto na `main`.
- [ ] **Documentação e fundação Antigravity:** `AGENTS.md`, `CLAUDE.md`, `nestjs-project/AGENTS.md`, `nestjs-project/CLAUDE.md`, `.agents/rules/nestjs-project.md` e `docs/diagrams/software-arch.mermaid` atualizados e 100% coerentes com o código real; fundação `.agents/` preservada e íntegra.

### 7.2. Verificação Negativa (Zero Itens de Reprova Automática)

- [ ] **NÃO** pulou nenhuma etapa do workflow (`research → plan-context → plan-validate → plan-resolve → plan-build → implement`).
- [ ] **NÃO** deixou `validation.md` em `dirty` nem omitiu SIs ou `Technical Specifications`.
- [ ] **NÃO** passou o arquivo de vídeo de até 10GB por dentro da API NestJS no upload.
- [ ] **NÃO** usou mock/filesystem falso no lugar de MinIO, fila Redis e worker reais no `compose.yaml`.
- [ ] **NÃO** há erro de `tsc --noEmit`, erro de `npm run lint` ou teste falhando.
- [ ] **NÃO** há nenhum commit direto na branch `main`.
- [ ] **NÃO** há nenhuma divergência entre `AGENTS.md`/`CLAUDE.md` e os arquivos/rotas reais do repositório.
