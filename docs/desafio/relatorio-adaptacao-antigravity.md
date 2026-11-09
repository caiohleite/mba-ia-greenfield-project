# Relatório de Adaptação e Migração: Claude Code → Google Antigravity

> **Projeto:** StreamTube (`mba-ia-greenfield-project`)  
> **Escopo:** Migração aditiva da configuração de agente (`.claude/`, `CLAUDE.md`, `.mcp.json`) para os padrões nativos do Google Antigravity (`AGENTS.md`, `.agents/`)  
> **Política de Retrocompatibilidade:** Zero exclusões (100% dos arquivos originais do Claude Code preservados intactos)

---

## 1. Resumo Executivo

Este documento registra a auditoria completa (Fase 0), a conversão técnica (Fases 1 a 4), a verificação sintática e estrutural (Fase 5) e as recomendações operacionais (Fase 6) da adaptação deste repositório — originalmente configurado para o **Claude Code** — para o **Google Antigravity**, seguindo estritamente:

- A especificação aberta **Agent Skills** ([https://agentskills.io/specification](https://agentskills.io/specification)) para todas as skills em `.agents/skills/<nome>/SKILL.md`.
- A documentação oficial do **Google Antigravity** ([https://antigravity.google/docs/overview](https://antigravity.google/docs/overview)) para regras (`AGENTS.md` e `.agents/rules/*.md`), servidores MCP (`.agents/mcp_config.json`) e subagents (`.agents/agents/*.md`).

---

## 2. Matriz Geral de Migração (Fase 0 — Auditoria → Destino Antigravity)

| Origem (Claude Code) | Tipo de Recurso | Destino (Antigravity) | Ação |
| :--- | :--- | :--- | :--- |
| `CLAUDE.md` (raiz) | Regra/contexto global (`always_on`) | `AGENTS.md` (raiz) | **Convertido** (original mantido) |
| `nestjs-project/CLAUDE.md` | Contexto de subprojeto (Backend) | `.agents/rules/nestjs-project.md` | **Convertido** (`trigger: glob`, `globs: "nestjs-project/**"`) |
| `next-frontend/CLAUDE.md` | Contexto de subprojeto (Frontend) | `.agents/rules/next-frontend.md` | **Convertido** (`trigger: glob`, `globs: "next-frontend/**"`) |
| `.claude/rules/*.md` (17 arquivos) | Regras modulares por caminho (`paths:`) | `.agents/rules/*.md` (17 arquivos) | **Convertido** (`trigger: glob` + `globs:` em CSV) |
| `.claude/skills/<nome>/` (23 skills) | Agent Skills (`SKILL.md` + recursos) | `.agents/skills/<nome>/` (23 skills) | **Convertido** (adequado à spec Agent Skills + progressive disclosure) |
| `.claude/agents/*.md` (6 arquivos) | Subagents customizados de leitura | `.agents/agents/*.md` (6 arquivos) | **Convertido** (referências internas apontadas para `.agents/`) |
| `.mcp.json` | Configuração de servidores MCP (`postgres`) | `.agents/mcp_config.json` | **Convertido** (schema `mcpServers` preservado integralmente) |
| `.mcp.json.example` | Exemplo documental MCP (`postgres`, `figma`) | — | **Mantido** (referência documental) |
| `.claude/commands/*.md` | Slash commands | `.agents/skills/<nome>/` | **N/A** (diretório inexistente; workflows já estão em `.agents/skills/` e tornam-se slash commands automaticamente no Antigravity) |
| `.claude/settings.json` → `hooks.UserPromptSubmit` | Hook de submissão de prompt | `AGENTS.md` (`## Agent Skill Usage`) | **Não mapeável 1:1** (comportamento já coberto nativamente pela regra `always_on` em `AGENTS.md`) |
| `.claude/settings.json` → `enabledPlugins["figma@claude-plugins-official"]` | Plugin oficial do marketplace Claude Code | — | **Não mapeável** (exclusivo do Claude Code; ver alternativa via MCP Store / `.agents/mcp_config.json`) |
| `.claude/settings.local.json` | Configurações locais não versionadas | — | **N/A** (inexistente no workspace) |
| `.claude/hooks/` | Diretório de scripts de hooks | — | **N/A** (inexistente no workspace) |
| `skills-lock.json` | Lockfile de instalação de skills externas | — | **Mantido** intacto |
| `_claude-sessions/*.txt` | Logs históricos de sessões | — | **Mantido** intacto |

---

## 3. Detalhamento por Fase

### Fase 1 — Skills (`.agents/skills/<nome>/SKILL.md`)

Todas as **23 skills** de `.claude/skills/` foram copiadas para `.agents/skills/` e validadas contra os requisitos da especificação [agentskills.io/specification](https://agentskills.io/specification):

1. **Frontmatter YAML válido** delimitado por `---` na linha 1, seguido do corpo em Markdown.
2. **`name`:** Obrigatório, $\le 64$ caracteres, apenas letras minúsculas, números e hífens (`^[a-z0-9]+(-[a-z0-9]+)*$`), sem iniciar/terminar com hífen, sem hífens consecutivos, e idêntico ao nome da pasta pai. Todos os 23 nomes originais já atendiam à regra (nenhuma renomeação de pasta foi necessária).
3. **`description`:** Obrigatória, não vazia, $\le 1024$ caracteres, descrevendo claramente **o que** a skill faz e **quando** o agente deve ativá-la. Em `generate-test-guide/SKILL.md`, o caminho citado na `description` foi atualizado de `.claude/skills/testing-guide-<project>/` para `.agents/skills/testing-guide-<project>/`.
4. **Campos opcionais e remoção de chaves proprietárias:**
   - Chaves proprietárias do Claude Code fora da especificação Agent Skills foram removidas das cópias em `.agents/skills/`: `disable-model-invocation: true` (presente em 16 skills) e `user-invocable: false` (presente em `next-best-practices`).
   - Os campos `license` e `metadata` (mapa `string → string`) presentes em `nestjs-best-practices` e `vercel-react-best-practices` foram preservados conforme a especificação.
   - O campo experimental `allowed-tools` em `playwright-cli/SKILL.md` teve sua sintaxe convertida do padrão Claude Code (`Bash(playwright-cli:*) Bash(npx:*) Bash(npm:*)`) para a sintaxe de permissões do Antigravity separada por espaços: `command(playwright-cli*) command(npx*) command(npm*)`.
5. **Progressive Disclosure ($\le 500$ linhas no `SKILL.md`):**
   - As 6 skills cujo `SKILL.md` ultrapassava 500 linhas tiveram seus blocos estáticos de referência (schemas JSON, templates de saída e bibliotecas de boilerplate) extraídos para subarquivos em `references/`, mantendo instruções explícitas de leitura sob demanda no corpo do `SKILL.md`:
     - `figma-audit-tokens/SKILL.md` (664 → 439 linhas): extraído `references/audit-json-schema.md` (snippets de captura de estilos da Step 4, regras de normalização da Step 5e e schema JSON completo da Step 6).
     - `plan-rule-author/SKILL.md` (654 → 454 linhas): extraído `references/boilerplate-and-skeleton.md` (biblioteca de boilerplates por dispatch point, esqueleto do arquivo gerado e procedimento manual para novo dispatch point).
     - `implement/SKILL.md` (592 → 442 linhas): extraído `references/frontend-screen-si-flow.md` (sub-passos detalhados de `SI-NN.X.0` Drift Audit e `SI-Xa` Visual Shell).
     - `screen-inventory/SKILL.md` (515 → 461 linhas): extraído `references/subagent-prompt-template.md` (template de prompt para os subagents extratores de tela).
     - `plan-context/SKILL.md` (506 → 345 linhas): extraído `references/context-output-template.md` (template Markdown completo de `context.md`).
     - `plan-phase/SKILL.md` (505 → 362 linhas): extraído `references/plan-phase-template.md` (template estrutural completo do documento de fase).

#### Catálogo Completo das 23 Skills Migradas

| # | `name` | Linhas (`SKILL.md`) | `description` Completa |
| :--- | :--- | :---: | :--- |
| 1 | `decide` | 224 | `Front-door for free-text decision needs. Triages user input against existing technical decisions, then routes to one of three branches: Revision (parameter change in existing decided TD), Supersede (Option letter change requires new TD), or Greenfield (no relevant TD exists). Trigger phrases: 'tem decisão sobre X?', 'quero mudar Y', 'preciso decidir Z', 'já decidimos sobre...', 'mudou o caminho/parâmetro/threshold de...', any free-text input describing a need that may or may not be covered by a TD. For structured phase research, use /research <phase> directly. For pipeline-stage operations on known scopes, use /plan-context, /plan-validate, /plan-resolve, /plan-build directly.` |
| 2 | `figma-apply-tokens-tailwind-v4` | 59 | `Apply a Figma → CSS audit report to a Tailwind v4 project. Reads docs/figma-audit.json produced by figma-audit-tokens, validates the CSS file hasn't changed since audit, applies drift fixes and inserts missing tokens following Tailwind v4 conventions (@theme inline for primitives, theme selector blocks for multi-mode tokens), and runs scale-gap analysis. Invoke ONLY when the user explicitly asks to apply, sync, or commit the audit — phrases like "apply figma audit", "sync figma tokens to CSS", "aplica os tokens do figma", "atualiza o CSS com o audit". Requires docs/figma-audit.json; if missing, instruct the user to run figma-audit-tokens first.` |
| 3 | `figma-audit-tokens` | 439 | `Audit drift between a Figma variable collection and a project's CSS variables. Captures Figma variables (all modes) plus text/effect styles, parses the project's CSS variables wherever they live (@theme inline, :root, .dark, custom selectors), and emits a structured JSON report at docs/figma-audit.json with drift items, missing-in-CSS items, and raw style observations. Framework-agnostic for variable drift; styles are emitted raw for downstream apply skills to analyze. Invoke ONLY when the user explicitly asks to audit, compare, or check drift between Figma and CSS — phrases like "audit figma tokens", "figma vs CSS drift", "compare figma to CSS variables", "puxa o estado do figma", "compara tokens do figma com o css". Never trigger on ambient Figma mentions or design-to-code work — those belong to figma-implement-design.` |
| 4 | `generate-test-guide` | 459 | `Analyzes a project's tech stack, searches the web for testing best practices, asks the user clarifying questions, and generates a project-specific testing multi-file skill at .agents/skills/testing-guide-<project>/ with main SKILL.md and artifact/reference sub-files. Invoke with /generate-test-guide <project-folder>.` |
| 5 | `implement` | 442 | `Execute a phase or task implementation plan step by step, respecting dependencies, running the relevant tests after each SI, and only advancing when tests pass. Use whenever the user asks to implement, execute, build, or deliver a planned phase or task — including variations like 'implement phase X', 'execute phase-02', 'build the auth phase', 'implement task <slug>', 'run the task plan', 'implement the SIs', or any request to turn a plan document (docs/phases/phase-NN-{slug}/phase-NN-{slug}.md or docs/tasks/task-{slug}/task-{slug}.md) into working code.` |
| 6 | `implement-phase` | 177 | `Execute a phase implementation plan step by step, respecting dependencies, running the relevant tests after each SI, and only advancing when tests pass. Use whenever the user asks to implement, execute, build, or deliver a planned phase — including variations like 'implement phase X', 'execute phase-02', 'build the auth phase', 'run the phase plan', 'implement the SIs', or any request to turn a phase plan document (docs/phases/phase-NN-*.md) into working code.` |
| 7 | `nestjs-best-practices` | 139 | `NestJS architecture patterns and best practices. TRIGGER when: planning or implementing NestJS features (modules, controllers, services, etc), designing modules, implementing auth/security, or writing NestJS tests. DO NOT TRIGGER when: only reading NestJS code without intent to modify, working on frontend, or asking general TypeScript questions.` |
| 8 | `next-best-practices` | 152 | `Next.js best practices - file conventions, RSC boundaries, data patterns, async APIs, metadata, error handling, route handlers, image/font optimization, bundling` |
| 9 | `plan-build` | 291 | `Stage 4 of the plan pipeline (phase and task modes). Runs abort-fast preflight checks (mode detection, context.md + validation.md present and fresh, status clean, library-refs.md present if required, resume detection) and writes the final phase-NN-{slug}.md or task-{slug}.md artifact directly from the main thread. The procedure is split across companion files: phase-a.md (scaffold + Technical Specifications + pause), phase-b.md (Step Implementations + Dependency Map + Deliverables), phase-c.md (append-mode for incremental delta builds). The pause between Phase A and Phase B creates a natural review checkpoint. Use as the final stage after validation.md reports clean. Triggers: 'plan-build NN', 'plan-build <slug>', 'build the phase NN', 'build task <slug>'.` |
| 10 | `plan-context` | 345 | `Stage 1 of the plan pipeline (phase and task modes). Consolidates project-plan.md (phase mode), decisions docs (phase-scope + ad-hoc tied to the phase, or the task's own + correlated docs), prior phases (or latest completed phase) and testing guide into a lean context.md. Use when the user asks to start planning a phase or task, generate the context, or run the first stage of the planning pipeline. Triggers: 'plan-context NN', 'plan-context <slug>', 'plan-context "prose"', 'gera o contexto da fase NN', 'inicia planejamento da task <slug>'.` |
| 11 | `plan-phase` | 362 | `Generate a technical implementation plan for a project phase. Use whenever the user asks to plan, detail, or generate the technical plan for a phase — including variations like 'plan phase X', 'detail the phase', 'generate phase implementation', 'create the phase-XX.md', or any mention of creating the technical document for a project phase.` |
| 12 | `plan-pipeline` | 355 | `Entry point and shared conventions for the plan pipeline (phase and task modes). Invoke directly when the user asks about planning without specifying a stage — this skill orients them to the pipeline stages (context → validate → resolve → build → specs [optional]). For actual work, use the stage skills: plan-context, plan-validate, plan-resolve, plan-build, plan-test-specs.` |
| 13 | `plan-resolve` | 353 | `Stage 3 of the plan pipeline (phase and task modes). Reads validation.md, asks the user via AskUserQuestion, and applies the answers to the decisions doc (filling **Decision:** fields, injecting superseded markers, or creating the decisions doc on-the-fly in task-sem-research), context.md (patching the Decisions Index), and validation.md (marking issues resolved). Dispatches Context7 for newly decided libraries and writes library-refs.md. In phase mode, aborts on MD-N with a /research instruction. In task mode, MD-N may be resolved inline by creating the decisions doc. Use after /plan-validate <arg> reports status: dirty. Triggers: 'plan-resolve NN', 'plan-resolve <slug>', 'resolve issues da fase NN', 'apply decisions to task <slug>'.` |
| 14 | `plan-rule-author` | 454 | `Scaffolds a new custom rule MD file under docs/rules/{plan-validate,plan-build,plan-resolve}/ following the canonical contract documented inline in this SKILL.md. Use when the user asks to create, author, or scaffold a new validation/build/resolve rule, register a new IC prefix, or add a custom check that fires under one of /plan-validate, /plan-build, or /plan-resolve. Trigger phrases: 'crie uma regra de plan-validate/build/resolve', 'nova regra de validação/build/resolve', 'add a custom rule that detects X', 'register a new IC prefix', 'scaffold a rule for {dispatch point}'. Out of scope: editing/disabling existing rules (Read+Edit directly), creating a new dispatch point (rare manual procedure documented in this SKILL.md as a tail section), authoring suppressions.md (owner-managed file).` |
| 15 | `plan-test-specs` | 437 | `Stage 5 of the plan pipeline (post-build). Generates and syncs <subproject>/specs/<scenario>.plan.md files from phase/task plans, in Microsoft spec-driven format. Specs are later consumed by /implement Step 3a — frontend path loads the playwright-cli Skill for Playwright pattern reference and LLM-authors the E2E test file; backend path LLM-authors its E2E test file (file path/suffix per the per-subproject testing-guide skill; what-to-test and best practices per artifact come from that same skill, already loaded at /implement Step 2). Use after /plan-build completes. Triggers: 'plan-test-specs NN', 'plan-test-specs <slug>', 'gerar test specs da fase X'.` |
| 16 | `plan-validate` | 418 | `Stage 2 of the plan pipeline (phase and task modes). Reads context.md (which embeds ## Decisions Detail and ## Inherited Decisions Detail with Recommendation + Libraries per TD), detects inconsistencies, ambiguities, missing decisions, dependency gaps (phase mode), and inherited-constraint conflicts. Produces or regenerates validation.md with a status: clean\|dirty verdict. Use after /plan-context <arg> and after each /plan-resolve <arg> cycle. Triggers: 'plan-validate NN', 'plan-validate <slug>', 'valida a fase NN', 'run validation stage'.` |
| 17 | `playwright-cli` | 388 | `Automate browser interactions, test web pages and work with Playwright tests.` |
| 18 | `research` | 433 | `Research technical options and generate a structured decisions document. Use whenever the user needs to explore alternatives, understand trade-offs, or define technical paths — whether for a project phase or an ad-hoc topic. Trigger phrases include: 'research options for phase X', 'research how to do Y', 'technical decisions for Z', 'what are the options for...', 'explore alternatives for...', 'research the phase', 'investigate options for parallel DB testing', or any mention of exploring alternatives, trade-offs, or technical choices before planning or implementing.` |
| 19 | `screen-inventory` | 461 | `Generate a screen inventory for a project phase or task. Infers which screens need to be built or updated from the phase's capabilities (phase mode) or from the task's scope prose (task mode), asks the user for the corresponding Figma URLs, then extracts each screen's components via the Figma MCP, classifies them by behavior (Presentational / Local-interactive / Server-connected), and maps server-connected components to capabilities or scope. Use whenever the user asks to inventory screens, extract components from Figma for planning, or prepare the front-end inputs before /plan-context — including variations like 'inventariar as telas', 'inventário de telas da fase NN', 'screen inventory NN', 'screen inventory <task-slug>', 'extrair componentes do Figma', or any mention of connecting Figma screens to planning before plan-context runs. Accepts a phase number (integer) OR a task slug (string), optionally with an upfront list of Figma URLs.` |
| 20 | `testing-guide-nestjs-project` | 140 | `Testing guide for nestjs-project. Reference this skill when planning features, implementing code, creating tests, or reviewing changes in nestjs-project. Covers what to test, at which layer, and how to set up each test — organized by artifact type. Triggers on: planning nestjs-project features, implementing nestjs-project features, writing tests for nestjs-project, reviewing nestjs-project code, reviewing nestjs-project tests, what should I test in nestjs-project, how to test nestjs-project, nestjs-project test guide.` |
| 21 | `testing-guide-next-frontend` | 137 | `Testing guide for next-frontend. Reference this skill when planning features, implementing code, creating tests, or reviewing changes in next-frontend. Covers what to test, at which layer, and how to set up each test — organized by artifact type. Triggers on: planning next-frontend features, implementing next-frontend features, writing tests for next-frontend, reviewing next-frontend code, reviewing next-frontend tests, what should I test in next-frontend, how to test next-frontend, next-frontend test guide.` |
| 22 | `typeorm` | 71 | `TypeORM patterns and database development guidelines. TRIGGER when: planning or implementing features involving database, TypeORM entities, repositories, migrations, queries, or data source configuration. DO NOT TRIGGER when: only reading database code without intent to modify, working on frontend, or asking general TypeScript questions.` |
| 23 | `vercel-react-best-practices` | 149 | `React and Next.js performance optimization guidelines from Vercel Engineering. This skill should be used when writing, reviewing, or refactoring React/Next.js code to ensure optimal performance patterns. Triggers on tasks involving React components, Next.js pages, data fetching, bundle optimization, or performance improvements.` |

---

### Fase 2 — Regras e Contexto (`AGENTS.md` e `.agents/rules/*.md`)

1. **`CLAUDE.md` (raiz) → `AGENTS.md` (raiz):**
   - Criado `AGENTS.md` na raiz do workspace (modo `always_on` nativo do Antigravity, sem frontmatter). Todo o conteúdo instrutivo (visão geral, arquitetura C4, Docker Networking, princípios de trabalho, Definition of Done, convenções Git Flow, política de testes, limites de escopo, ativação de skills e consulta obrigatória ao Context7) foi preservado integralmente.
2. **Consolidação de `CLAUDE.md` em subpastas (`nestjs-project/CLAUDE.md` e `next-frontend/CLAUDE.md`):**
   - Como o Antigravity centraliza as regras modulares do workspace em `.agents/rules/`, os dois arquivos de subdiretório foram convertidos em regras acionadas por glob em `.agents/rules/`:
     - `.agents/rules/nestjs-project.md` (`trigger: glob`, `globs: "nestjs-project/**"`)
     - `.agents/rules/next-frontend.md` (`trigger: glob`, `globs: "next-frontend/**"`)
3. **Regras modulares (`.claude/rules/*.md` → `.agents/rules/*.md`):**
   - As 17 regras modulares existentes em `.claude/rules/` tiveram seu frontmatter convertido do formato `paths:` (lista YAML do Claude Code) para o formato do Antigravity (`trigger: glob` e `globs:` em string CSV).

#### Catálogo Completo das 19 Regras em `.agents/rules/`

| # | Arquivo em `.agents/rules/` | Modo (`trigger`) | Padrão (`globs`) | Tamanho |
| :--- | :--- | :---: | :--- | :---: |
| 1 | `auth-jwt.md` | `glob` | `nestjs-project/src/auth/**` | 3.530 B |
| 2 | `nestjs-common-conventions.md` | `glob` | `nestjs-project/src/**/*.ts` | 1.712 B |
| 3 | `nestjs-controllers.md` | `glob` | `nestjs-project/**/*.controller.ts` | 5.888 B |
| 4 | `nestjs-dtos.md` | `glob` | `nestjs-project/**/*.dto.ts` | 3.197 B |
| 5 | `nestjs-entities.md` | `glob` | `nestjs-project/**/*.entity.ts` | 1.281 B |
| 6 | `nestjs-layer-separation.md` | `glob` | `nestjs-project/**/*.controller.ts, nestjs-project/**/*.guard.ts, nestjs-project/**/*.interceptor.ts, nestjs-project/**/*.pipe.ts, nestjs-project/**/*.filter.ts` | 2.763 B |
| 7 | `nestjs-modules.md` | `glob` | `nestjs-project/src/**/*.module.ts` | 1.371 B |
| 8 | `nestjs-project.md` | `glob` | `nestjs-project/**` | 8.170 B |
| 9 | `nestjs-services.md` | `glob` | `nestjs-project/**/*.service.ts` | 2.589 B |
| 10 | `nestjs-testing.md` | `glob` | `nestjs-project/**/*.spec.ts, nestjs-project/**/*.integration-spec.ts, nestjs-project/**/*.e2e-spec.ts, nestjs-project/test/**` | 5.479 B |
| 11 | `next-frontend-bff-api.md` | `glob` | `next-frontend/app/api/**/route.ts, next-frontend/lib/api/**/*.ts` | 5.161 B |
| 12 | `next-frontend-code-quality.md` | `glob` | `next-frontend/**/*.ts, next-frontend/**/*.tsx` | 4.834 B |
| 13 | `next-frontend-msw-mocks.md` | `glob` | `next-frontend/mocks/**, next-frontend/**/*.integration.test.ts, next-frontend/**/*.integration.test.tsx` | 6.822 B |
| 14 | `next-frontend-testing.md` | `glob` | `next-frontend/**/__tests__/**, next-frontend/tests/**, next-frontend/**/*.test.ts, next-frontend/**/*.test.tsx, next-frontend/**/*.integration.test.ts, next-frontend/**/*.integration.test.tsx, next-frontend/**/*.e2e-spec.ts` | 1.953 B |
| 15 | `next-frontend-ui.md` | `glob` | `next-frontend/components/**/*.tsx, next-frontend/app/**/*.tsx, next-frontend/app/**/*.css` | 7.898 B |
| 16 | `next-frontend.md` | `glob` | `next-frontend/**` | 18.856 B |
| 17 | `typeorm-migrations.md` | `glob` | `nestjs-project/**/migrations/**, nestjs-project/**/*data-source.ts` | 3.315 B |
| 18 | `typeorm-queries.md` | `glob` | `nestjs-project/src/**/*.service.ts, nestjs-project/src/**/*.repository.ts` | 2.896 B |
| 19 | `typescript-strict.md` | `glob` | `nestjs-project/src/**/*.ts, nestjs-project/test/**/*.ts` | 2.234 B |

---

### Fase 3 — Servidores MCP (`.agents/mcp_config.json`)

O arquivo `.mcp.json` na raiz do projeto foi convertido para `.agents/mcp_config.json` mantendo exatamente o comando, argumentos e variáveis do servidor `postgres`:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://streamtube:streamtube@localhost/streamtube"
      ]
    }
  }
}
```

**Candidatos da MCP Store / Configuração Adicional Identificados:**
- **`context7`**: exigido pela seção `## Library Documentation Lookup` em `AGENTS.md` e utilizado pelas skills `research` e `plan-resolve`. Pode ser habilitado diretamente pela **MCP Store** do Antigravity.
- **`figma`**: referenciado em `.mcp.json.example` (`http://127.0.0.1:3845/mcp`) e em `.claude/settings.json` (`enabledPlugins`). Pode ser habilitado pela **MCP Store** do Antigravity ou adicionado em `.agents/mcp_config.json` como servidor Streamable HTTP (`"url": "http://127.0.0.1:3845/mcp"`).

---

### Fase 4 — Comandos, Subagents, Hooks e Settings

1. **Slash Commands (`.claude/commands/`):**
   - O projeto não utilizava `.claude/commands/`; todos os comandos procedurais (`/decide`, `/research`, `/screen-inventory`, `/plan-context`, `/plan-validate`, `/plan-resolve`, `/plan-build`, `/plan-test-specs`, `/plan-rule-author`, `/implement`, `/implement-phase`, `/generate-test-guide`, `/figma-audit-tokens`, `/figma-apply-tokens-tailwind-v4`) já estavam implementados como skills e foram migrados para `.agents/skills/`, ficando automaticamente disponíveis no Antigravity.
2. **Subagents Customizados (`.claude/agents/*.md` → `.agents/agents/*.md`):**
   - Os 6 subagents especializados do pipeline de planejamento (`decisions-correlator.md`, `decisions-detail-reader.md`, `decisions-reader.md`, `inventory-digest-reader.md`, `phases-reader.md`, `plan-reader.md`) foram migrados para `.agents/agents/` com suas referências internas atualizadas para `.agents/`.
3. **Hooks e Settings (`.claude/settings.json`):**
   - Veja a seção 4 abaixo para o detalhamento dos itens não mapeáveis diretamente (`hooks.UserPromptSubmit` e `enabledPlugins`).

---

## 4. Itens Não Mapeáveis e Alternativas Sugeridas

1. **Hook `UserPromptSubmit` em `.claude/settings.json`**
   - **Configuração original:**
     ```json
     "hooks": {
       "UserPromptSubmit": [
         {
           "hooks": [
             {
               "type": "command",
               "command": "echo 'Identify and activate relevant skills for this task.'"
             }
           ]
         }
       ]
     }
     ```
   - **Motivo da não mapeabilidade direta:** No Claude Code, qualquer texto impresso em `stdout` por um hook `UserPromptSubmit` é concatenado como texto livre ao prompt. No Antigravity, o sistema de hooks (`.agents/hooks.json`, evento `PreInvocation`) exige um payload JSON estruturado em `stdout` (`{"injectSteps": [{"systemMessage": {"systemMessage": "..."}}]}`).
   - **Alternativa aplicada:** A seção `## Agent Skill Usage` em `AGENTS.md` já contém essa instrução de forma mais completa e é injetada automaticamente em todos os turnos (`always_on`), cumprindo 100% do objetivo sem custo de spawn de processo a cada mensagem.

2. **Plugin `enabledPlugins["figma@claude-plugins-official"]` em `.claude/settings.json`**
   - **Motivo da não mapeabilidade:** Identificador exclusivo do registro interno de plugins do Claude Code.
   - **Alternativa sugerida:** Configurar o servidor MCP do Figma via **MCP Store** do Antigravity ou incluir o endpoint Streamable HTTP (`"figma": { "url": "http://127.0.0.1:3845/mcp" }`) em `.agents/mcp_config.json`.

3. **Campos proprietários `disable-model-invocation` e `user-invocable` nos `SKILL.md`**
   - **Motivo da não mapeabilidade:** São extensões proprietárias do Claude Code não previstas na especificação aberta [agentskills.io/specification](https://agentskills.io/specification).
   - **Alternativa aplicada:** Removidos das cópias em `.agents/skills/*/SKILL.md` para conformidade estrita com a especificação; as regras de ativação permanecem governadas pelo campo `description` (ex.: `"Invoke ONLY when the user explicitly asks..."`).

---

## 5. Alterações Mínimas Inevitáveis Realizadas nas Cópias (`.agents/`)

- **Extração para `references/` por Progressive Disclosure:** 6 arquivos `SKILL.md` em `.agents/skills/` que excediam 500 linhas (`figma-audit-tokens`, `plan-rule-author`, `implement`, `screen-inventory`, `plan-context`, `plan-phase`) tiveram suas seções de templates/schemas extraídas para arquivos em `references/`, mantendo links e instruções explícitas de leitura no `SKILL.md`.
- **Normalização de `allowed-tools` em `.agents/skills/playwright-cli/SKILL.md`:** Convertido de `Bash(playwright-cli:*) Bash(npx:*) Bash(npm:*)` para `command(playwright-cli*) command(npx*) command(npm*)`.
- **Conversão de Frontmatter nas Regras em `.agents/rules/*.md`:** Substituição de `paths:` (lista YAML) por `trigger: glob` + `globs:` (string CSV).
- **Atualização de Caminhos Internos nas Cópias em `.agents/`:** Referências internas a `.claude/skills/`, `.claude/rules/`, `.claude/agents/`, `${CLAUDE_PLUGIN_ROOT:-.}/.claude/skills/` e `global CLAUDE.md` foram atualizadas para `.agents/skills/`, `.agents/rules/`, `.agents/agents/` e `global AGENTS.md`.

---

## 6. Verificação Final (Fase 5)

- **Validação YAML/JSON:** Todos os 23 arquivos `.agents/skills/<nome>/SKILL.md`, os 19 arquivos `.agents/rules/*.md`, os 6 arquivos `.agents/agents/*.md` e `.agents/mcp_config.json` foram validados sintaticamente via parser YAML/JSON.
- **Integridade de Referências Internas:** Todos os caminhos relativos (`references/*`, `scripts/*`, `rules/*`, `artifacts/*`, `templates/*`, `phase-*.md`, `testing-fundamentals.md`, `slicing-implementation-plan.md`) referenciados dentro das skills existem e resolvem corretamente em `.agents/skills/`.
- **Confirmação da Restrição 3 (Zero Exclusões):** O comando `git status --short` confirma que nenhum arquivo existente do projeto ou do Claude Code foi removido ou modificado — apenas `AGENTS.md` e `.agents/` foram adicionados.

---

## 7. Recomendações de Próximos Passos Manuais (Ações do Usuário)

> [!IMPORTANT]
> As ações abaixo dependem de preferências de ambiente local e política de segurança do usuário, e portanto **não foram executadas automaticamente**.

1. **Habilitar os servidores MCP `context7` e `figma` no Antigravity:**
   - Abra **Additional Options (`...`) > MCP Servers** (ou a **MCP Store**) no Antigravity e habilite o **Context7** (exigido por `AGENTS.md` para consulta de documentação de bibliotecas) e o **Figma MCP**.
   - Alternativamente, caso utilize o servidor local do Figma Desktop na porta `3845`, adicione a entrada abaixo em `.agents/mcp_config.json`:
     ```json
     {
       "mcpServers": {
         "postgres": {
           "command": "npx",
           "args": [
             "-y",
             "@modelcontextprotocol/server-postgres",
             "postgresql://streamtube:streamtube@localhost/streamtube"
           ]
         },
         "figma": {
           "url": "http://127.0.0.1:3845/mcp"
         }
       }
     }
     ```
2. **Configurar Permissões de Ferramentas e MCP (`/permissions`):**
   - Como o suporte ao campo experimental `allowed-tools` varia por implementação, recomenda-se registrar as permissões equivalentes no gerenciador `/permissions` do Antigravity (escopo **Project** ou **Global**):
     - **Comandos (`allow`):**
       - `command(playwright-cli*)`
       - `command(npx*)`
       - `command(npm*)`
     - **Servidores MCP (`allow`):**
       - `mcp(postgres/*)`
       - `mcp(context7/*)`
       - `mcp(figma/*)`
