# Frontend Screen SI Flow (SI-NN.X.0 Drift Audit & SI-Xa Visual Shell)

**SI-NN.X.0 — Drift audit (precedes every SI-Xa):** load `figma:figma-implement-design` plugin skill. The audit's purpose is to surface DS-component drift to the user via `frontend-drift-report.md` BEFORE SI-Xa applies any DS file edits. The audit RECORDS decisions; it does NOT apply edits. Schema reference: `.agents/skills/plan-build/references/frontend-drift-report-schema.md`.

Sub-steps in order:

**1. Read prior sections of `frontend-drift-report.md`.** The file lives at `docs/phases/phase-NN-{slug}/frontend-drift-report.md` (sibling of plan doc) when this is not the first audit-SI of the phase. If the file exists, parse all `## Screen: ...` sections and build a `prior_decisions` map keyed by `component_path → (status, decision, source_si)`. If the file does not exist (this is the first audit-SI of the phase), `prior_decisions` is empty and the audit-SI will CREATE the file in sub-step 5.

**2. Read this screen's UI Contract — Reused DS list + Figma URL.** The SI-NN.X.0 block carries `**Figma:**` (URL) and `**UI Contract:**` (pointer to `#### Screen: <name>` subsection). Bounded-read the screen subsection (locate via `Grep -n '^#### Screen: '`, pick the cited screen, upper bound = next `^#### ` or `^### ` line). Pull the **`**Reused DS components:**` bullet list verbatim** — the rich form `<path> [(new)?] — <usage hint>`. Each path is canonical (e.g., `components/ui/button.tsx`); the `(new)` marker is documentation-only — at audit-SI runtime the file always exists on disk because bootstrap SIs (`SI-NN.0.k`) ran first per Dependency Map (B2.6 Step 5 wires bootstrap → audit-SI dependencies).

**3. Invoke `figma:figma-implement-design` with the narrow handoff.** Same handoff shape as SI-Xa (Decisão #31 — no auth, no errors, no validation):

- Figma URL (from `**Figma:**` field)
- Reused DS list (path + usage hint)
- Server-connected component **names only**
- Target paths (inferred from route + framework convention) — read-only context for the audit; no writes happen here

**Screenshot discipline:** pass `excludeScreenshot: true` to `get_design_context` when a separate `get_screenshot` call follows. One image per audit-SI invocation, not two.

The plugin returns design context (structured representation) and a screenshot. The reference snippet's `<Component prop='value-A' ...>` calls form the **demand list** consumed by sub-step 4 — the same demand list shape SI-Xa used to consume in the pre-audit-SI model.

**4. Classify each Reused DS component per the 4-value status enum + populate Decision per default policy.** For each component in the Reused DS list, read the file from disk and diff against the Figma demand.

**Before composing `auto-Edit "<specifics>"` Decisions, consult the project's design-system alias map** (one-time per `/implement` session — memoized, same pattern as library-refs.md):

- `Bash test -f .agents/rules/design-system.md` (typical project path; adjust per project convention).
- If present: `Grep -n '^## .* Aliases' .agents/rules/design-system.md` to locate the aliases section → bounded-Read it (upper bound: next `^## ` heading or end of file). Parse the table into an in-memory map: `(figma_value, component_path) → code_variant`. Component scope `*` matches all paths.
- If absent (no rule file, or no aliases section): alias map is empty; audit proceeds — additive fallback applies.

Apply the discriminator from the schema reference:

- `alinhado` → byte-level match between DS value and Figma value (same Tailwind utility, custom-property reference, or mathematical equivalence)
- `drift menor` → deviation within DS token system, same token family/category (e.g., `rounded-xl` vs `rounded-md`)
- `drift relevante` → missing surface (variant / prop / state demanded but undeclared), system breakage (different token family), hardcoded leak (hex/rgb literal), or component file absent on disk
- `componente ausente` → file listed in Reused DS list does not exist on disk

For `drift menor` / `drift relevante`, compose `auto-Edit "<specifics>"` per the **3-form constraint** in schema § "Naming constraint for `auto-Edit` `<specifics>`" — try in order; first match wins:

1. **Exact-name retune** — DS file has a variant with the same name as the Figma demand value → emit `auto-Edit "retune <prop>: <existing-variant> — <old> → <new>"`.
2. **Alias-mapped retune** — alias map contains `(figma-name, *)` (global) OR `(figma-name, <this-component-path>)` (per-component) → emit `auto-Edit "retune <prop>: <code-variant> per Figma '<figma-name>' demand — <old> → <new>"`.
3. **Additive fallback** — neither exact nor alias matches → emit `auto-Edit "+<dim> '<figma-name>' from Figma demand"`. The `<figma-name>` is **byte-literal** from the Figma component property; **inventing names from screen context (slug, route, feature, business domain) is forbidden** per schema § Naming constraint.

Default decision per status (see schema § Default decisions):

- `alinhado` → `skip`
- `drift menor` → `auto-Edit "<specifics>"` (per 3-form constraint above)
- `drift relevante` → `auto-Edit "<specifics>"` (default per 3-form constraint above — try retune first, fall back to additive; cross-screen consultation below may upgrade to CONFLICT prefix — CONFLICT default verb body is **form 3 only** (additive) per schema § Decision row, since forms 1 + 2 retune existing variants and would break the prior screen)
- `componente ausente` → `create`

**Cross-screen consultation (CONFLICT detection).** For each Reused DS component, look up its `component_path` in `prior_decisions`:

- If absent → first-time component; standard discriminator flow; `**Prior:**` is `_(none)_`.
- If present AND prior decision was `exception "<reason>"` → re-classify standalone (no auto-Edit was applied; file in original state); `Prior: "exception at <source_si> — '<reason>'"` (informational only, no CONFLICT framing).
- If present AND current Figma demand reproduces prior outcome → `status: alinhado`, `decision: skip`, `Prior: "<prior summary> at <source_si> honored"`.
- If present AND current Figma demand DIVERGES from prior outcome → `status: drift relevante`; verb is `CONFLICT` rendered in the vertical hierarchical format per schema § Decision:

  ```markdown
  - **Decision:** `CONFLICT`
    - **Note:** <one-liner describing divergence — e.g., "current screen demands `size: lg`, prior demanded `size: md`">
    - **Verb body:** `auto-Edit`
      - +<dim> '<figma-name>' from Figma demand
  ```

  `Prior: "<prior summary> at <source_si>"`. **Default `**Verb body:**` is `auto-Edit` form 3 only** (additive — new variant identity); forms 1 + 2 (retune existing or alias-mapped) are structurally unsafe in CONFLICT cases because they modify variants the prior screen depends on. **The naming constraint from sub-step 4 (and schema § Naming constraint) still applies** — `<figma-name>` is byte-literal from the current screen's Figma component property; inventing names from screen context is forbidden. User reviews and may override at the file-based checkpoint (e.g., switch the `**Verb body:**` to a retune form, accepting the prior-screen-break risk).

**Variant-conflict scan.** After deciding the base-form drift bullets above, run a second pass: for every property the auto-Edit bullets target (e.g., `background`, `border`, `border-width`, `radius`, `padding`, `gap`, `text-size`, `font-weight`), grep the DS file for sibling utilities of the SAME property under any Tailwind variant prefix. Variant prefixes come in two shapes that BOTH need coverage: (a) **concise** — single-word or hyphenated identifiers like `dark:`, `hover:`, `focus:`, `focus-visible:`, `disabled:`, `aria-invalid:`, `data-checked:`; (b) **bracket-arbitrary** — Tailwind v4's `data-[<attr>=<val>]:` / `aria-[<attr>=<val>]:` forms like `data-[state=open]:`, `data-[size=sm]:`, `aria-[expanded=true]:`. Variant prefixes can also be **chained** (e.g., `dark:hover:`, `dark:data-[state=checked]:`) up to a few levels deep. A regex covering all three shapes: `(?:^|\s)((?:[\w-]+(?:\[[^\]]+\])?:){1,3})(bg|border|rounded|p[xytrbl]?|gap|text|font)-` — breakdown: `[\w-]+` matches the identifier (concise + hyphenated), `(?:\[[^\]]+\])?` optionally captures a bracketed arbitrary value, `:` terminates each prefix, `{1,3}` allows chaining.

For each sibling found, classify:

- **Stale shadcn/library opinion that conflicts with the project's token-driven control** — e.g., DS file has `bg-transparent ... dark:bg-input/30` and the audit is moving `bg-transparent` → `bg-input-background` (a project token explicitly designed to cover both modes via cascade). The `dark:` variant has higher specificity (`0,2,0` vs `0,1,0`) and would override the new base utility under `.dark`. Emit an additional bullet: `drop variant override: <variant>:<prop>-<value> (conflicts with token-driven <prop>)`.
- **Legitimate state utility that drifts from Figma's resolved value in the same state** — e.g., DS has `hover:bg-muted` but Figma's hover state demands a different fill. Emit: `retune variant override: <variant>:<prop>: <old> → <new>`.
- **Legitimate state utility that does NOT drift** — leave alone; do not emit a bullet.

The classification is project-aware: read `globals.css` (or the project's design-system rule file) to know which custom properties are dual-mode tokens (have light + dark resolutions). When such a token exists for the property the audit is tuning, **`dark:`-style mode-covering variants on that property are stale by definition** — the project chose the token paradigm; the cascade already covers the dark mode via the token, so the `dark:<prop>-X` utility is residual coupling from the library scaffold (typical with `npx shadcn add`) that overrides the token via specificity. Emit `drop variant override:` for those. **State variants (`hover:`, `focus:`, `focus-visible:`, `aria-*:`, `data-*:`, etc.) are NOT stale** — they scope a state, not a mode the token covers; classify them per the retune-vs-skip path above (retune if value drifts from Figma's same-state demand, skip if it matches).

These bullets are **outside the 3-form Figma-name-anchored constraint** (they target stale code-side opinions, not Figma demand). See schema § "Variant-conflict bullets" for their canonical form.

**5. Write the section to `frontend-drift-report.md`.** Compose a `## Screen: <slug> — audited at SI-NN.X.0 ({YYYY-MM-DD})` section per the schema § Body. The section is **vertical, NOT tabular**: it consists of a `**Quick scan:**` tally line, a TOC bullet list (one bullet per component), and one `### {path} — {ComponentName}` H3 section per component with `**Status:**` / `**Decision:**` (with nested specifics bullets per the hybrid intelligent format) / `**Prior:**` fields.

**Hybrid intelligent format rules (per schema § Decision):**

- `skip` → inline (no nested bullets).
- `auto-Edit` → bulleted; one specifics bullet per drift dimension (≥1; multi-dimension drift → multi-bullet).
- `exception` → bulleted with one `reason:` bullet.
- `create` → bulleted with one `source:` bullet.
- `CONFLICT` → bulleted with `**Note:**` bullet (audit's diagnostic) + `**Verb body:**` sub-section nesting its own verb + specifics. CONFLICT default verb body is `auto-Edit` form 3 only (additive).

Compose the TOC bullets deterministically from the H3 sections per the suffix rules in schema § Body "TOC format". The `Quick scan` tally is the count of each Status enum value.

Two write cases:

- **First audit-SI of the phase** (file does not exist) → CREATE the file with frontmatter (`kind: drift-report`, `phase: phase-NN-{slug}`, `plan_mtime: <ISO timestamp captured at this run>`) + H1 `# phase-NN-{slug} — Drift Report` + this screen's section (Quick scan + TOC + H3 sections). Mark "Created: frontend-drift-report.md." in the per-SI completion report.
- **Subsequent audit-SI** (file exists) → APPEND this screen's full section in execution order; OR overwrite-in-place if this same audit-SI ran before (re-run; date in heading reflects last run; TOC + H3 sections re-emitted from current state). Preserve existing frontmatter — `plan_mtime` is immutable per schema § Frontmatter immutability and is NOT recaptured on subsequent runs. Mark "Updated: frontend-drift-report.md." in the per-SI completion report.

**No code edits in the target subproject.** AC enforcement: `git diff --name-only HEAD -- <target-subproject>` after the SI must be empty. The plan folder lives outside `<target-subproject>` so the report file is excluded by path scope.

**6. Pause for user review at Step 6 (default mode).** The user opens `frontend-drift-report.md`, optionally edits Decisions on per-component H3 sections (the file-based override mechanism — see schema § File-based user override mechanism), saves, and responds "Seguir". `/implement` then proceeds to SI-NN.Xa, which reads the (potentially user-edited) report and applies decisions verbatim.

---

**SI-Xa — Visual shell (≤2 Technical actions; depends on SI-NN.X.0):** load `figma:figma-implement-design` plugin skill. SI-Xa is now SIMPLIFIED — drift detection moved to the audit-SI; SI-Xa applies the audit's decisions mechanically and renders the screen. The constraint relaxation from "1 Technical action only" to "≤2 Technical actions" is an intentional template change to accommodate the new 2-action shape (apply drift decisions + visual shell generation).

Sub-steps in order:

**1. Read the Drift Report section for this screen.** The SI-Xa block carries a `**Drift Report:** see \`frontend-drift-report.md\` → \`## Screen: <slug>\`` field. Bounded-read that section — locate via `Grep -n '^## Screen: <slug>'` on the report file; upper bound = next `^## Screen: ` or end of file.

The section is the **vertical hierarchical format** per schema § Body — Quick scan + TOC + N H3 sections (one per component). Iterate H3 sections within the bounded read; the TOC bullet list is auto-generated by the audit and **ignored by SI-Xa** (parser uses H3 sections as the source of truth).

**Per-component parsing algorithm (within the bounded screen section):**

```
For each `^### {path} — {ComponentName}` heading found within the bounded screen section:
  - bound this H3's region: lines from this `### ` heading until the next `^### ` or `^## ` or end of bounded screen section
  - extract `path` and `ComponentName` from the H3 line
  - within H3's region, extract:
    - status_line: line matching `^- \*\*Status:\*\* <value>` → status enum value
    - decision_line: line matching `^- \*\*Decision:\*\* \`<verb>\`` → 5-verb enum value
    - decision_specifics: lines indented `^  - <text>` directly after decision_line, until non-2-space-indented line OR another `^- \*\*` field
    - prior_line: line matching `^- \*\*Prior:\*\* <value>` → informational only
  - dispatch by verb (see step 2)
```

Backslash-escape `**` in regex (markdown bold markers); use ripgrep regex on the bounded read.

**2. Apply Decisions verbatim — mechanical application; no judgment.** For each H3 component section parsed:

- `skip` → no-op (specifics block is empty/inline by construction)
- `auto-Edit` → for **each** specifics bullet (one per drift dimension), apply one `Edit` on the cited DS file using the bullet text per the 3-form constraint in schema § "Naming constraint". Multi-bullet → multi-Edit on the same file
- `exception` → no-op (read `reason:` bullet only for logging in the per-SI completion report; preserves existing DS value)
- `create` → use the `source:` bullet to determine asset/spec. For icons (asset name/role recorded by audit), re-fetch via `figma:figma-implement-design` in sub-step 3 below if asset URL is needed at create time, OR pull from the design context returned by sub-step 3 once it has run. For component files (custom-business-component), create per UI Contract spec
- `CONFLICT` → ignore the `**Note:**` bullet (informational diagnostic, audit-owned). Find the `**Verb body:**` bullet, extract its inner verb (`auto-Edit` or `exception` only on CONFLICT rows — `create`/`skip` categorically incompatible), and recursively apply that verb's nested specifics (auto-Edit's specifics bullets, OR exception's reason bullet)

**No drift detection or auto-judgment in this step.** The audit-SI did the analysis; SI-Xa is mechanical application. If a component's H3 section is malformed (verb not in 5-verb enum, verb incompatible with Status, missing required nested bullet for verb shape, CONFLICT without `**Note:**` or `**Verb body:**`), **halt and ask the user to fix the report** (do not improvise — the report is the contract).

**Path-resolution fallback (rename corner case).** If a path the Drift Report cites doesn't exist on disk (a previous SI-Xa renamed/split the file), grep the DS component directory for a component whose default export name matches the cited file's stem. On exactly one match, treat that as the resolved path and proceed. On zero or multiple matches, **halt and ask the user**.

**Post-edit variant-conflict guard (defense-in-depth).** After applying every `auto-Edit` bullet that introduces or retunes a base-form (no variant prefix) Tailwind utility on property X (e.g., `bg-input-background`, `rounded-md`, `border border-border`), grep the same DS file for sibling utilities of property X under any variant prefix — both **concise** (`dark:`, `hover:`, `focus:`, `disabled:`, `data-checked:`, etc.) and **bracket-arbitrary** (`data-[state=open]:`, `aria-[expanded=true]:`, etc.), including **chained** forms (e.g., `dark:hover:`, `dark:data-[state=checked]:`) — using the same regex shape from the audit-SI's variant-conflict scan. If a sibling is found AND the Drift Report's H3 section for this component does NOT contain a `drop variant override:` or `retune variant override:` bullet for it: **halt and ask the user** with a one-liner like `"<file>: applying base utility 'bg-input-background' but found sibling 'dark:bg-input/30' uncovered by Drift Report — variant has higher specificity (0,2,0) and would override base in dark mode. Drop, retune, or keep?"`. The user resolves with one of: (a) drop the sibling (apply an extra Edit removing it from the class string), (b) retune the sibling (apply an extra Edit changing its value), (c) keep the sibling (proceed; the user accepts the override behavior). The chosen action is recorded in the per-SI completion report's `Updated existing DS:` annotation. This guard is the defense-in-depth complement to the audit-SI's variant-conflict scan — catches drift that the audit missed (e.g., a variant prefix the audit's regex didn't enumerate, or a project where the dual-mode token paradigm wasn't legible from `globals.css`).

**3. Visual shell generation — invoke `figma:figma-implement-design` with the narrow handoff (Decisão #31).** Pass:

- Figma URL (from SI-Xa `**Figma:**` field)
- Reused DS list (path + usage hint) — *now reflecting the DS edits applied in sub-step 2*
- Server-connected component **names only**
- Target paths (inferred from route + framework convention)

**Do NOT pass** Auth requirement, Rendering strategy, Error Catalog mapping, Client-side validation mirror, or endpoint references — those concerns belong to SI-Xb.

**Screenshot discipline:** same as audit-SI — pass `excludeScreenshot: true` to `get_design_context` when a separate `get_screenshot` call follows.

Translate the reference snippet into project-shaped code using the (now-aligned) DS components. Write `<page.tsx>` and `<form-component.tsx>`.

**Anti-pattern — NO call-site override of intrinsic visual identity.** Allowed via call-site overrides: utilities that **position the component relative to its parent** (full-width fill, margin, flex grow/shrink, gap, alignment). Forbidden: any utility / style / inline rule that re-defines the component's intrinsic visual identity (dimensions, typography, color, border, radius, background) — the fix for those belongs at the DS file via the audit's `auto-Edit` rows, not at the call site.

**SI-Xa completion report extension.** When the audit's Decisions list `auto-Edit` / `create` rows for DS files, SI-Xa applies them. The per-SI completion report (Step 6 below) carries an `Updated existing DS:` line listing each modified DS file with a one-token annotation matching the verb body (`(+variant '<name>')`, `(+variant '<name>' (retuned))`, `(+prop '<name>')`, `(+state '<name>')`, `BREAKING(prop): <one-liner>`, `BREAKING(rename): <one-liner>`). For `BREAKING(rename)` cases, caller files patched as a side effect go on `Updated:` (not `Updated existing DS:` — callers are not DS components). See Step 6 for the canonical shape.

After SI-Xa completes, **stop invoking** `figma:figma-implement-design` for the rest of the screen's work (SI-Xb + onward). This is the cadencing mechanism — not literal skill unload, but behavioral: SI-Xa loads Figma MCP data during its single invocation; MCP releases data when invocation returns; SI-Xb runs without re-invoking figma plugin → no accumulated Figma context. Framework-specific sub-skills (`next-auth-guard`, `react-hook-form-validation`, etc.) load fresh into SI-Xb scope without MCP pressure.

