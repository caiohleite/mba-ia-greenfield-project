# Plan Context — Output Format Template

## Output format

```markdown
---
kind: phase | task
name: phase-NN-{slug} | task-{slug}
sources_mtime:
  docs/project-plan.md: "ISO-8601-timestamp"                    # phase mode only
  docs/decisions/technical-decisions-{slug}.md: "ISO-8601-timestamp"   # if exists
  # one line per ad-hoc decisions doc contributing via related_phases (phase mode)
  # one line per correlator-confirmed decisions doc (both modes)
  # one line per prior-phase context.md read via phases-reader
  # one line per testing-guide skill file read
---

# {name} — Context

## Scope

**Phase mode** — verbatim from plan-reader output:

**Phase name:** ...
**Capabilities** (literal, `docs/project-plan.md`):

- ...

**Out of scope:** ...
**Deliverables:** ...
**Affected subprojects:** ...
**Deferred subprojects:** ...
**Sequencing notes:** ...

**Neighbors (for boundary detection only):**

- **Phase {NN-1}:** ...
- **Phase {NN+1}:** ...

**Task mode** — `scope_prose` verbatim (no sub-fields):

> {the prose from inline input or scope_description}

## Decisions Index

_(from decisions-reader — one row per TD across phase-scope + ad-hoc docs)_

| Ref | Source | Scope | Topic | Status | Decision | Libraries |
|-----|--------|-------|-------|--------|----------|-----------|
| {slug}/TD-01 | phase | Backend | ... | pending | — | — |
| ...

_Source files:_

- {slug} — `docs/decisions/technical-decisions-{slug}.md` (scope_type: phase)
- ... (ad-hoc files)

## Capability Coverage

| Capability (from project-plan.md) | Covered by |
|-----------------------------------|------------|
| <capability bullet verbatim> | {slug}/TD-01, {slug}/TD-03 |
| <capability bullet verbatim> | — _(no TD yet — plan-validate will flag as MD)_ |

_Build this table by cross-referencing the `Capability:` field of each TD (available to decisions-reader via bounded grep) with the capabilities from plan-reader. If a TD's Capability is `Transversal — covers: ...`, list it under every covered capability. If a capability has no TD, leave "—"._

## Decisions Detail

_(current-phase TDs only — from decisions-detail-reader)_

### {slug}/TD-01

**Recommendation:** {prose stripped of "Option X — " prefix}
**Libraries:** lib-a, lib-b

### {slug}/TD-02

**Recommendation:** {prose}
**Libraries:** —

## Inherited Decisions Detail

_(inherited TDs from prior phases — from phases-reader, dedupe applied)_

### prior-slug/TD-01

**Recommendation:** {prose}
**Libraries:** —

### prior-slug/TD-05

**Recommendation:** {prose}
**Libraries:** typeorm, pg

## Inherited Conventions

_(from phases-reader — compact list; sourced from prior phases in phase mode, from the latest completed phase in task mode)_

- <convention bullet, ≤150 chars> _(from phase MM)_
- ...

## Inherited Deferred Capabilities

_(from phases-reader — informational-only; plan-validate does NOT fire issues based on unaddressed entries)_

| Capability | Status | Origin phase | Rationale |
|-----------|--------|--------------|-----------|
| "Tela de histórico" | deferred | phase-03-my-videos | Escopo reduzido pra entrega inicial |

_(or `_No inherited deferred capabilities._` when empty)_

## UI Inventory

_(from inventory-digest-reader — present only when UI scope detected)_

**Source:** `docs/inventories/screen-inventory-phase-NN-{slug}.md` _(or `docs/tasks/task-{slug}/inventory.md` in task mode)_
**Screens in scope:** N

### UI ↔ Capability Join

| Screen | Route | Verb | Capability | Covering Component |
|--------|-------|------|------------|-------------------|
| ... | ... | ... | ... | ... |

### Server-connected Components

- `{ComponentName}` ({Screen}) — `Reuse?: {path | new}` _(per `screen-inventory/SKILL.md` Output Contract item 4: existing path, OR bare literal `new` for pure-DOM. The third form `<path> (new)` is normalized to bare `new` by `inventory-digest-reader` for Server-connected emissions — see that agent's Procedure step 5.)_
- ...

### Open Questions from Inventory

_(verbatim bullets from the inventory's `## Open questions` section; ingested by plan-validate as OQ-N)_

- ...

### Inherited UI Components

_(task mode only; omitted when empty)_

- `{ComponentName}` — `Reuse?: {path}` (from phase-MM-{slug}) _(only already-implemented components: real paths with no `(new)` suffix — `inventory-digest-reader` excludes the planned `<path> (new)` form from inheritance per its Procedure step 7.4)_

## Non-UI / Deferred Capabilities

_(always emitted in final context.md; placeholder `_None._` when empty. Omitted in partial context.md per Decisão #28. Read by plan-context/plan-build; write-append by plan-resolve.)_

| Capability | Status | Rationale | TD refs |
|-----------|--------|-----------|---------|
| (empty on first assembly — plan-resolve appends rows as user marks capabilities) |

## Testing Requirements

_(from testing-guide skill(s) loaded in step 4)_

### {subproject}

| Artifact type | Required layers |
|---------------|-----------------|
| ... | ... |

_(repeat per subproject in scope)_

### {deferred subproject, if any}

_Deferred subproject — testing requirements will be defined when the testing-guide skill is created._
```

