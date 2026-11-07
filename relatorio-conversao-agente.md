# Relatório de Conversão de Agente: Claude Code para Google Antigravity & AgentSkills.io

Este documento detalha a análise, o plano de migração e cada ajuste executado para converter a estrutura de agentes de IA do repositório **StreamTube** do padrão proprietário do **Claude Code** para a especificação aberta **[AgentSkills.io](https://agentskills.io/home)** e o sistema de customizações do **Google Antigravity**.

> **Escopo respeitado:** Nenhum arquivo de código-fonte da aplicação (`nestjs-project/src/`, `nestjs-project/test/`, configurações de build/Docker ou documentos de arquitetura em `docs/`) foi modificado. Apenas os artefatos de configuração de agentes de IA foram convertidos.

---

## 1. Análise Inicial e Pontos de Ajuste Identificados

Na estrutura original voltada ao Claude Code, foram mapeados os seguintes elementos que exigiam conversão:

| Artefato Original (Claude Code) | Artefato Convertido (Antigravity / AgentSkills.io) | Motivo da Conversão |
| :--- | :--- | :--- |
| `CLAUDE.md` (raiz) | `AGENTS.md` (raiz) | O Antigravity descobre regras hierárquicas de diretório automaticamente através de arquivos `AGENTS.md` (ou `GEMINI.md`). |
| `nestjs-project/CLAUDE.md` | `nestjs-project/AGENTS.md` | Regra de escopo de subdiretório para o projeto backend NestJS, contendo também referência à `Bash tool` que precisava ser adaptada para `run_command`. |
| `.claude/rules/*.md` (9 arquivos) | `.agents/rules/*.md` (9 arquivos) | O Claude Code utiliza `paths:` (lista YAML) no frontmatter, enquanto o Antigravity exige `trigger: glob` e `globs:` (string com padrões separados por vírgula). |
| `.claude/skills/*` (7 skills) | `.agents/skills/*` (7 skills) | O diretório de descoberta de workspace no Antigravity é `.agents/skills/`. Além disso, 4 skills continham frontmatter proprietário (`disable-model-invocation: true`) e chamadas a ferramentas exclusivas do Claude Code (`AskUserQuestion`, `WebSearch`, `Skill`, `Write`, `Edit`, `TaskCreate`, `TaskUpdate`). |
| `.claude/settings.json` | `.agents/hooks.json` | O Claude Code configura hooks em `settings.json` com o evento `UserPromptSubmit` e saída em texto puro. O Antigravity utiliza `.agents/hooks.json` com hooks nomeados, evento `PreInvocation` e contrato estrito de saída JSON (`injectSteps`). |
| `.mcp.json` | `.agents/mcp_config.json` e `.agents/plugins/streamtube-mcp/` | O Antigravity utiliza `mcp_config.json` (no escopo de workspace/plugin ou global) em vez de `.mcp.json`, além de exigir o servidor MCP `context7` referenciado nas regras e skills do projeto. |

---

## 2. Plano Passo a Passo Executado

### Passo 1: Migração das Regras de Diretório (`CLAUDE.md` → `AGENTS.md`)
1. **`AGENTS.md` (raiz):**
   - Criado a partir de `CLAUDE.md`, atualizando o cabeçalho principal para `# AGENTS.md`.
   - Mantidas integralmente todas as instruções de visão geral, arquitetura C4, networking Docker, convenções Git Flow, política de testes, limites de escopo, uso de skills e consulta obrigatória de documentação via MCP `context7`.
2. **`nestjs-project/AGENTS.md`:**
   - Criado a partir de `nestjs-project/CLAUDE.md`, atualizando o cabeçalho para `# AGENTS.md`.
   - Na seção `## Long-running Processes`, a instrução `must be run in background in the Bash tool` foi ajustada para `must be run in background in the run_command tool (using IsDaemon: true)`, refletindo a ferramenta nativa de execução de comandos em background do Google Antigravity.
3. Remoção dos arquivos legados `CLAUDE.md` e `nestjs-project/CLAUDE.md`.

---

### Passo 2: Conversão das Regras Modulares (`.claude/rules/` → `.agents/rules/`)
Foram migradas as 9 regras modulares do projeto para `.agents/rules/`, convertendo o frontmatter YAML do padrão Claude Code:

```yaml
# Antes (Claude Code)
---
paths:
  - 'nestjs-project/**/*.controller.ts'
description: 'Controller conventions...'
---
```

Para o padrão oficial do Google Antigravity:

```yaml
# Depois (Google Antigravity)
---
trigger: glob
globs: "nestjs-project/**/*.controller.ts"
description: "Controller conventions..."
---
```

**Arquivos convertidos em `.agents/rules/`:**
1. `nestjs-common-conventions.md` (`globs: "nestjs-project/src/**/*.ts"`)
2. `nestjs-controllers.md` (`globs: "nestjs-project/**/*.controller.ts"`)
3. `nestjs-dtos.md` (`globs: "nestjs-project/**/*.dto.ts"`)
4. `nestjs-entities.md` (`globs: "nestjs-project/**/*.entity.ts"`)
5. `nestjs-layer-separation.md` (`globs: "nestjs-project/**/*.controller.ts, nestjs-project/**/*.guard.ts, nestjs-project/**/*.interceptor.ts, nestjs-project/**/*.pipe.ts, nestjs-project/**/*.filter.ts"`)
6. `nestjs-modules.md` (`globs: "nestjs-project/src/**/*.module.ts"`)
7. `nestjs-services.md` (`globs: "nestjs-project/**/*.service.ts"`)
8. `nestjs-testing.md` (`globs: "nestjs-project/**/*.spec.ts, nestjs-project/**/*.e2e-spec.ts, nestjs-project/test/**"`)
9. `typeorm-migrations.md` (`globs: "nestjs-project/**/migrations/**"`)

---

### Passo 3: Adequação das Skills ao Padrão AgentSkills.io e Google Antigravity (`.claude/skills/` → `.agents/skills/`)

Todas as 7 skills foram migradas para `.agents/skills/<skill-name>/` e revisadas frente à especificação **AgentSkills.io** (`https://agentskills.io/specification`):

#### 3.1. Conformidade de Frontmatter e Diretórios (AgentSkills.io)
- **Campos obrigatórios (`name` e `description`):** Verificados em todas as 7 skills:
  - `name` utiliza apenas caracteres alfanuméricos minúsculos e hifens (`a-z`, `0-9`, `-`), possui menos de 64 caracteres e corresponde exatamente ao nome do diretório pai.
  - `description` é não-vazia, possui menos de 1024 caracteres e descreve claramente **o que** a skill faz e **quando** o agente deve ativá-la.
- **Campos opcionais (`license`, `metadata`):** Mantidos em `nestjs-best-practices/SKILL.md` conforme a especificação AgentSkills.io.
- **Remoção de campos não-padrão:** Removida a chave `disable-model-invocation: true` do frontmatter de `generate-test-guide`, `implement-phase`, `plan-phase` e `research`. Além de não fazer parte da especificação base do AgentSkills.io, essa diretiva impedia que o agente ativasse automaticamente as skills por decisão semântica (conforme instruído em `AGENTS.md` na seção *Agent Skill Usage*).

#### 3.2. Adaptações Mínimas no Corpo das Skills para o Google Antigravity
1. **`generate-test-guide/SKILL.md`:**
   - Caminhos atualizados de `.claude/skills/` para `.agents/skills/` e de `.claude/rules/` para `.agents/rules/`.
   - Ferramenta `WebSearch` substituída por `search_web`.
   - Ferramenta `AskUserQuestion` substituída por `ask_question`.
   - Menção específica ao modelo `Claude` substituída por `the agent`.
2. **`plan-phase/SKILL.md`:**
   - Substituída a instrução `use the Skill tool to load it` por leitura direta sob demanda (`load it by reading .agents/skills/testing-guide-{subproject}/SKILL.md`), seguindo o mecanismo de *Progressive Disclosure* do Antigravity.
   - Substituídas as referências às ferramentas `Write`, `Edit` e `AskUserQuestion` por `write_file`, `edit_file` e `ask_question`.
   - Ajustada a referência ao tool call do Context7 de `mcp__context7__query-docs` (nomenclatura interna do Claude Code) para `Context7 query-docs MCP tool`.
3. **`implement-phase/SKILL.md`:**
   - Substituída a instrução `use the Skill tool to load testing-guide-{subproject}` por `load testing-guide-{subproject} (by reading .agents/skills/testing-guide-{subproject}/SKILL.md)`.
   - Substituídas as chamadas às ferramentas `TaskCreate` e `TaskUpdate` (exclusivas do Claude Code) pelo controle de estado por SI (`pending` → `in_progress` → `completed`) diretamente no arquivo persistente de progresso (`docs/phases/phase-NN-*.progress.md`) utilizando `write_file` e `edit_file`.
4. **`research/SKILL.md`:**
   - Removido `disable-model-invocation: true` do frontmatter; corpo já utilizava instruções agnósticas compatíveis com `search_web` e Context7 MCP.
5. **`testing-guide-nestjs-project/SKILL.md` (+ 12 arquivos em `artifacts/` e 4 em `references/`):**
   - Migrados integralmente. Estrutura multi-arquivo com carregamento sob demanda (lazy loading) 100% aderente ao AgentSkills.io e Antigravity.
6. **`nestjs-best-practices/SKILL.md` (+ 42 arquivos em `rules/`):**
   - Migrados integralmente. Removida apenas a última linha órfã que apontava para um arquivo compilado inexistente (`AGENTS.md`) dentro da skill para evitar conflito com o `AGENTS.md` da raiz do projeto.
7. **`typeorm/SKILL.md` (+ 16 arquivos em `rules/`):**
   - Migrados integralmente sem necessidade de alterações no corpo.

---

### Passo 4: Conversão de Lifecycle Hooks (`.claude/settings.json` → `.agents/hooks.json`)
O arquivo `.claude/settings.json` definia um hook `UserPromptSubmit`:
```json
{
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
}
```
No Google Antigravity, os hooks ficam em `.agents/hooks.json`, são organizados por um identificador nomeado no topo do objeto JSON, utilizam o evento `PreInvocation` (executado antes de cada chamada ao modelo) e exigem que o comando escreva um objeto JSON válido no `stdout` contendo `injectSteps`. A configuração convertida em `.agents/hooks.json` ficou:

```json
{
  "skill-activation-reminder": {
    "PreInvocation": [
      {
        "type": "command",
        "command": "printf '%s\\n' '{\"injectSteps\":[{\"ephemeralMessage\":\"Identify and activate relevant skills for this task.\"}]}'"
      }
    ]
  }
}
```

---

### Passo 5: Conversão da Configuração MCP (`.mcp.json` → `.agents/mcp_config.json` e `.agents/plugins/streamtube-mcp/`)
1. O arquivo `.mcp.json` original continha apenas o servidor `postgres`, embora `AGENTS.md`, `plan-phase` e `research` exijam o servidor MCP `context7`.
2. Foram configurados ambos os servidores (`postgres` e `context7` via `@upstash/context7-mcp`) em:
   - `.agents/mcp_config.json` (configuração MCP na raiz de customizações do workspace).
   - `.agents/plugins/streamtube-mcp/plugin.json` e `.agents/plugins/streamtube-mcp/mcp_config.json` (bundle de plugin de workspace do Antigravity, garantindo carregamento automático dos servidores MCP em qualquer superfície do Antigravity — CLI, IDE e App 2.0).

---

## 3. Funcionalidades do Claude Code Não Disponíveis Nativamente no Antigravity e Abordagens de Contorno

Durante a análise e conversão, foram identificadas as seguintes diferenças arquiteturais e funcionalidades específicas do Claude Code que não possuem equivalência direta 1:1 no Google Antigravity, juntamente com as soluções aplicadas:

### 3.1. Ferramentas de Todo-List de Sessão (`TaskCreate` / `TaskUpdate` / `TodoWrite`)
- **Como funciona no Claude Code:** O Claude Code possui ferramentas internas (`TaskCreate`, `TaskUpdate`, `TodoWrite`) para exibir um checklist de tarefas na interface do terminal durante a sessão, utilizadas intensivamente pela skill `implement-phase`.
- **Situação no Antigravity:** O Antigravity não possui as ferramentas `TaskCreate` ou `TaskUpdate` (a ferramenta `manage_task` do Antigravity destina-se exclusivamente ao gerenciamento de processos/comandos em background e cron jobs, e não a checklists de tarefas).
- **Abordagem de Contorno Aplicada:**
  1. **Persistência via `.progress.md` (já nativa da skill):** A skill `implement-phase` foi adaptada para registrar e atualizar o status de cada Step Implementation (`pending` → `in_progress` → `completed`) diretamente no arquivo `docs/phases/phase-NN-*.progress.md` usando `write_file` e `edit_file`.
  2. **Uso de Artifacts do Antigravity (opcional/complementar):** No Antigravity IDE / App 2.0, o agente também pode criar ou atualizar documentos Markdown no diretório de artefatos da conversa (`<appDataDir>/brain/<conversation-id>/`) para visualização rica de progresso pelo usuário.

### 3.2. Ferramenta Explícita `Skill` (`Skill tool`) vs. Progressive Disclosure via Leitura de Arquivo
- **Como funciona no Claude Code:** O Claude Code expõe uma tool específica chamada `Skill` que o modelo invoca para carregar uma skill no contexto.
- **Situação no Antigravity:** O Antigravity injeta automaticamente o `name`, `description` e o caminho de cada `SKILL.md` no prompt de sistema. Quando o agente decide ativar uma skill (ou quando uma skill instrui o carregamento de outra, como `plan-phase` e `implement-phase` carregando `testing-guide-<subproject>`), a ativação ocorre lendo o arquivo `.agents/skills/<name>/SKILL.md` através da ferramenta `read_file`.
- **Abordagem de Contorno Aplicada:** As instruções em `plan-phase/SKILL.md` e `implement-phase/SKILL.md` foram atualizadas para instruir a leitura direta de `.agents/skills/testing-guide-{subproject}/SKILL.md`.

### 3.3. Escopo de Descoberta de `mcp_config.json`
- **Como funciona no Claude Code:** O arquivo `.mcp.json` na raiz do repositório é lido diretamente pelo CLI.
- **Situação no Antigravity:** De acordo com a documentação do sistema de customizações do Antigravity, servidores MCP são carregados a partir da configuração global (`~/.gemini/config/mcp_config.json`) ou de plugins habilitados (`plugins/<plugin_name>/mcp_config.json`), além do suporte em `.agents/mcp_config.json` em superfícies integradas.
- **Abordagem de Contorno Aplicada:** Além de criar `.agents/mcp_config.json`, foi criado o plugin local de workspace `.agents/plugins/streamtube-mcp/` contendo `plugin.json` e `mcp_config.json`. Como plugins em `.agents/plugins/` são descobertos e habilitados por padrão no Antigravity, os servidores `postgres` e `context7` ficam imediatamente disponíveis para toda a equipe ao clonar o repositório, sem necessidade de configuração manual em `~/.gemini/config/mcp_config.json` (alternativamente, o desenvolvedor também pode registrá-los globalmente via `agy mcp add`).

### 3.4. Contrato de Hooks (`UserPromptSubmit` com stdout texto puro vs. `PreInvocation` com stdout JSON)
- **Como funciona no Claude Code:** O hook `UserPromptSubmit` aceita qualquer texto impresso no `stdout` (ex.: `echo '...'`) e o anexa ao prompt.
- **Situação no Antigravity:** Hooks no Antigravity comunicam-se via contrato estrito JSON (`protojson` em `camelCase`) tanto no `stdin` quanto no `stdout`. Saídas em texto puro causam erro de parsing.
- **Abordagem de Contorno Aplicada:** O hook foi migrado para `.agents/hooks.json` utilizando o evento `PreInvocation` e emitindo o payload JSON `{"injectSteps":[{"ephemeralMessage":"Identify and activate relevant skills for this task."}]}`, que injeta a mensagem efêmera no turno atual exatamente como ocorria no Claude Code.

---

## 4. Verificação Final de Conformidade

- [x] **Especificação AgentSkills.io:** Todas as 7 skills em `.agents/skills/` possuem `SKILL.md` com frontmatter YAML válido (`name` em kebab-case coincidindo com o diretório pai, `description` clara e < 1024 caracteres, sem chaves proprietárias inválidas).
- [x] **Padrões Google Antigravity:**
  - Regras de diretório na raiz (`AGENTS.md`) e no subprojeto (`nestjs-project/AGENTS.md`).
  - Regras modulares em `.agents/rules/*.md` com `trigger: glob`, `globs` e `description`.
  - Hooks em `.agents/hooks.json` seguindo o schema `PreInvocation` + `injectSteps`.
  - MCP configurado em `.agents/mcp_config.json` e `.agents/plugins/streamtube-mcp/mcp_config.json`.
- [x] **Integridade do Código da Aplicação:** Nenhum arquivo fora das configurações de agentes foi modificado.
