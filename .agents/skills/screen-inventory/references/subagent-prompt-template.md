# Screen Inventory — Sub-agent Prompt Template

### Sub-agent prompt template

Use this as the prompt body when spawning each sub-agent (fill in the bracketed fields):

```
You are inventorying a single Figma screen for the screen-inventory skill.

Screen: [screen name from the reconciled list]
Figma fileKey: [fileKey]
Figma nodeId: [nodeId in MCP form, e.g. 1234:5]
URL: [full URL]

Your task:
1. Read .agents/skills/screen-inventory/SKILL.md, sections "How to classify components", "How to derive verbs of intent", and "Output structure" (for the exact screen section format to emit). These are the rules you must follow.
2. Call mcp__plugin_figma_figma__get_design_context with the fileKey and nodeId above.
3. Call mcp__plugin_figma_figma__get_screenshot with the same.
4. List every component in the tree. Classify each as Presentational, Local-interactive, or Server-connected, using evidence from BOTH the Figma output AND the phase capabilities listed below.
5. For each Server-connected component, derive one or more verbs of intent and map each verb to exactly one capability from the list below (quote the capability verbatim).

Phase capabilities (verbatim from docs/project-plan.md, phase [NN]):
- "[capability 1]"
- "[capability 2]"
- ...

Already-classified components to reuse rather than re-deriving (aggregated by the parent from the current inventory file on resume AND from prior-phase inventories under docs/inventories/ when unchanged; empty on a fresh run with no cross-phase matches):
- [ComponentName] → [Type], In DS?: ✓/✗[, reuse path] (source: current | phase-NN)
- ...
(or the literal string "none")

Filesystem-existing DS paths (canonical `In DS?` source — built by the parent in `## Context` step 6 via `Glob <frontend-subproject>/components/**/*.{tsx,ts}`; this set is the ground truth for whether a path actually exists in-repo, NOT Figma's Code Connect map):
- [path]
- [path]
- ...
(or the literal string "none" when the components directory is empty or missing)

**Precedence rules for `In DS?` and `Reuse?`** (apply in this order; first match wins):

1. **Inherited list** — If a component appears in the "Already-classified components" list, inherit its `In DS?` and `Reuse?` values exactly as given — do NOT re-derive them from the Figma output, the Filesystem-existing DS paths set, or any inference about the codebase. If the record says `In DS?: ✗`, keep `✗` in your output even if a path seems to exist elsewhere. **For planned-but-not-yet-created components** (the second `Reuse?` form per Output Contract item 4 — `<path> (new)`), keep `In DS?: ✗` AND emit the path with the literal ` (new)` suffix preserved byte-verbatim — the suffix is the load-bearing detection signal for `phase-b.md` § B2.6 (bootstrap SI synthesis). Never strip it, never normalize it, never substitute the bare literal `new`.

2. **Filesystem-existing DS paths** — If the component is NOT in the inherited list AND Figma's `get_design_context` (Code Connect map) emits a `Reuse?` path candidate, check the path against the "Filesystem-existing DS paths" set above. **In set → emit the path verbatim with `In DS?: ✓`. NOT in set → emit the SAME path with the literal ` (new)` suffix appended AND `In DS?: ✗`.** This is the canonical mechanism that prevents Code Connect's lag-behind-reality from leaking into the inventory as false-positive `✓` rows (e.g., a Code Connect entry for `components/ui/button.tsx` while the file has not yet been authored on disk).

3. **No path emitted by Figma** — If Figma's output suggests no DS-path target for the component (genuinely new pure-DOM element like `<h1>`, `<p>`, helper text, inline link), emit the bare literal `new` (no path) with `In DS?: ✗`.

The parent, not you, is responsible for any updates to DS state beyond this discriminator (e.g., flipping a row's `In DS?` from `✗` to `✓` after a future phase actually creates the component — that propagates via the cross-phase reuse rule at Context step 4 on the next inventory run).

Return format:

Return ONLY a markdown block matching the screen section format defined in "Output structure" (no preamble, no explanation). Populate Component inventory, Verbs of intent, and Observations (omit Observations if empty). For any row you cannot resolve, use `[DECISION: option A | option B — concrete context including component name, node id, and what evidence is missing]` as the value in the ambiguous column. Never omit a row, never return a separate marker list.

Rules:
- Do NOT write to any file.
- Do NOT invent components not present in the Figma output.
- Do NOT define API contracts, endpoints, or HTTP specifics. Verbs are intents, not endpoints.
- Do NOT ask questions — return ambiguities as markers instead.
- Components visible only in the screenshot but absent from get_design_context MUST be flagged in the Observations subsection, not silently filled in.
```

