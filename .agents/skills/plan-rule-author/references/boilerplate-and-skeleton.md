# Plan Rule Author — Boilerplate Library, Body Skeleton & Dispatch Extension

## Per-dispatch boilerplate library

These blocks are auto-substituted into the generated rule body. Substitute `{PREFIX}` with `NAME.upper()` (the uppercased filename root from Step 2b — e.g., `openapi-consistency` → `OPENAPI-CONSISTENCY`) and `{accumulator_name}` with the value from Step 6 when applicable.

### Emission contract — plan-validate hard-fail

```markdown
## Emission

- **Prefix:** `{PREFIX}-N` (one IC per matching condition).
- **Severity:** hard-fail — flips `validation.md` `status: dirty`; entries go to `issues:` array (not `advisories:`).
- **Destination:** `validation.md` `issues:` accumulator (merged with built-in checks by the host's merge step).
```

### Emission contract — plan-validate advisory

```markdown
## Emission

- **Prefix:** `{PREFIX}-N` (one IC per matching condition).
- **Severity:** advisory — never blocks. Entries go to `validation.md` `advisories:` array; `status: dirty` is NOT flipped.
- **Destination:** `validation.md` `advisories:` accumulator.
```

### Emission contract — plan-build hard-fail

For build hard-fail rules, the body uses **two** Emission-family headings:

1. `## Emission contract` near the top — short prefix/severity/destination summary (block below).
2. `## Emission (build abort)` after Three-pass evaluation — the expanded sentinel-injection + abort-message sequence from Step 6.

This split exists because the abort flow is too verbose to nest inside the contract block, and readers benefit from seeing the contract upfront before the predicate/three-pass detail.

```markdown
## Emission contract

- **Prefix:** `{PREFIX}-N` (one per uncaptured finding in `{accumulator_name}`).
- **Severity:** hard-fail — aborts the host's render flow; no `<!-- phase-a-complete -->` is written.
- **Destination:** terminal `FAILED at step-4-{name}` message + `<!-- {name}-pending -->` sentinel injected in plan file.
```

### Emission contract — plan-build advisory

```markdown
## Emission

- **Prefix:** `{PREFIX}-N`.
- **Severity:** advisory — terminal log only; build proceeds. No sentinel injection.
- **Destination:** stdout (build log line).
```

### Emission contract — plan-resolve side-effect

```markdown
## Emission contract

- **Prefix:** Side-effect; does not emit IC.
- **Severity:** side-effect.
- **Destination:** {describe what the rule edits — decisions doc / context.md / library-refs.md}; appended summary line to host's next-command output.
```

### Suppressions — plan-validate (hard-fail or advisory)

```markdown
## Suppressions

1. Read `{target_dir}/suppressions.md` if present; no-op silently if absent. (`{target_dir}` is the host scope dir — `docs/phases/phase-NN-{slug}/` in phase mode or `docs/tasks/task-{slug}/` in task mode; resolved by the host from context.md `kind` + `name`.)
2. For each entry under `suppressions:` whose `id` matches `{PREFIX}-N`, add the ID to the per-rule suppressed set. Missing/malformed entries are skipped silently.
3. Filter emissions against the suppressed set. Honored IDs render in `validation.md`'s `## Active Suppressions` instead of `## Findings`.
```

### Suppressions — plan-build hard-fail

```markdown
## Suppressions

{PREFIX} aborts the build, but suppression is still honored:

1. Read `{target_dir}/suppressions.md` if present; no-op silently if absent.
2. For each entry whose `id` matches `{PREFIX}-N`, add the matching item to the suppressed set.
3. Filter `{accumulator_name}` against the suppressed set before deciding whether to abort. If `{accumulator_name}` becomes empty after filtering → no abort, no `<!-- {name}-pending -->` injection; the host proceeds to mark complete.
4. Even when filtering leaves `{accumulator_name}` non-empty and the abort proceeds, the user audits honored suppressions via `{target_dir}/suppressions.md` directly (no surfacing artifact at build time).
```

### Suppressions — plan-build advisory

```markdown
## Suppressions

Honored advisory suppressions have no surfacing artifact at build time:

1. Read `{target_dir}/suppressions.md` if present; no-op silently if absent.
2. For each entry whose `id` matches `{PREFIX}-N`, add the ID to the per-rule suppressed set.
3. Filter terminal-log emissions against the suppressed set. Honored IDs are silently dropped; the user audits via `{target_dir}/suppressions.md` directly.
```

### Suppressions — plan-resolve

```markdown
## Suppressions

Not applicable. Resolve rules are side-effect-only and emit no IC, so there is nothing to suppress. To prevent a side-effect from running, edit or remove the input that triggers it before re-invoking `/plan-resolve`.
```

## Generated body skeleton

After all steps, the rule MD has this exact structure (in this order):

```markdown
---
description: "{description from Step 2, ≤200 chars}"
status: active
---

# {HUMAN_TITLE}

`{HUMAN_TITLE}` MUST be human-readable and self-explanatory — never the bare kebab-case filename. Pattern: title-case rendering of `{name}`, optionally split with an em-dash (`—`) for semantic grouping when the name has a domain prefix + property tail. Technical tokens preserve their canonical casing (e.g., `OpenAPI`, `I18N`, `JWT`, not `Openapi` / `I18n` / `Jwt`). Examples:
- `# OpenAPI Consistency` ✅ (filename `openapi-consistency.md` — single-phrase rendering)
- `# I18N Sync — Missing Translation Keys` ✅ (filename `i18n-sync-missing-keys.md` — em-dash splits domain prefix `i18n-sync` from property tail `missing-keys`, expanded to a fuller phrase)
- `# openapi-consistency` ❌ (kebab-case literal; reader can't tell what this is)
- `# OC` ❌ (cryptic abbreviation; opaque)

{OPENING_PARAGRAPH}

`{OPENING_PARAGRAPH}` MUST be 2-3 sentences in plain language covering: (1) **what** the rule detects or applies (concrete, no jargon — "stale openapi.json files" not "F2-class drift"); (2) **when** it runs (which dispatch + which moment in the host's flow — "during /plan-validate after Checks 1-8" or "during /plan-build mid-render, right after the {input subsection} is emitted"); (3) **what action** it triggers (block / log advisory / edit files). Assume the reader has not read other rules and may be opening this file because they hit the `{PREFIX}-N` ID in validation.md and want to understand it. AVOID: forward references to other rules, references to frozen plans, undefined acronyms beyond `{PREFIX}` itself.

## Emission {contract}    ← per Step 7 boilerplate; the word "contract" appears only for build hard-fail (where the contract block is split from the abort sequence)

{boilerplate per dispatch + severity}

## Inputs

{INPUTS_PROSE from Step 4}

## Predicate

{PREDICATE_PROSE from Step 4}

### Pseudocode    ← optional; only if user opted in at Step 4'

{pseudocode in fenced code block, from Step 4'}

### Issue text    ← optional; only if user opted in at Step 4' (validate/build emission rules); for build hard-fail this lives after ## Emission instead

{literal IC text in fenced block, from Step 4'}

## Three-pass evaluation     ← always present; body is "Skipped. {reason}" when classifier said Skipped

{either "Skipped. {reason}" or the Apply block from Step 5}

## Emission (build abort)    ← only for build + hard-fail; insert Step 6 expanded sequence here

{Step 6 sequence}

## Suppressions

{Step 7 boilerplate}

## Depends on    ← only for resolve rules with inter-rule deps; from Step 6'

{predecessor list}

## Notes    ← optional; only if user opted in at Step 7' (Notes)

{notes prose from Step 7' — design rationale, trade-offs}

## Out of scope    ← optional; only if user opted in at Step 7' (Out of scope)

{bulleted list of out-of-scope items from Step 7'}

## Idempotency    ← optional; only if user opted in at Step 7' (Idempotency)

{idempotency guarantee prose from Step 7'}
```

Sections that are conditional:
- Three-pass when Skipped → still include with one-line "Skipped. {reason}" body.
- build-abort block → omit if not build+hard-fail.
- Depends on → omit if no resolve dependencies.
- **Pseudocode / Issue text (Predicate sub-blocks)** → omit unless opted in at Step 4'.
- **Notes / Out of scope / Idempotency (tail sections)** → omit unless opted in at Step 7'.


## Out of scope: adding a new dispatch point (rare procedure; documented here for reference)

This SKILL only authors rules under one of the three existing dispatch points (`plan-validate`, `plan-build`, `plan-resolve`). Adding a 4th dispatch point is a one-time architectural change that touches a host skill's source code and CANNOT be scaffolded by this SKILL — the procedure is documented here only so the knowledge does not become tribal.

**When you might need this:** you are introducing a new pipeline command (e.g., `/plan-something`) and want it to dispatch custom rules the same way `/plan-validate` does.

**Procedure:**

1. Pick the host skill and the dispatch step in its procedure (the moment when rules should fire).
2. `mkdir docs/rules/{skill-name}/` (e.g., `docs/rules/plan-something/`).
3. Add ~5-10 lines of glue to the host skill's procedure at the dispatch point. Canonical shape:

   ```markdown
   ## Custom rules

   At {dispatch step}, run all rules registered for /{skill-name}:

   1. `Glob docs/rules/{skill-name}/*.md`. If empty, skip.
   2. For each file in alphabetic order: bounded-Read frontmatter; if `status: disabled`, skip silently; otherwise full Read + follow body. Body declares emit prefix, severity, output destination.

   See `.agents/skills/plan-rule-author/SKILL.md` for the canonical rule contract.
   ```

4. (Optional) Update this SKILL.md's Step 1 question block (lines ~62-71) to list the new dispatch point as option 4, and add corresponding Emission contract + Suppressions blocks to the `## Per-dispatch boilerplate library` so future rules under the new point are scaffoldable.
5. Authors of subsequent rules under this dispatch point: zero edit to the host skill (Glob+Read picks them up automatically).

**Why this isn't automated:** adding a dispatch point is per-host-skill design work — choosing the right dispatch step, deciding emission destination semantics (does it write to a doc? abort the host? edit files?), and committing to a suppressions policy. Wizard-scaffolding it would either over-prescribe or under-specify; the manual procedure above is short enough that no automation is justified.
