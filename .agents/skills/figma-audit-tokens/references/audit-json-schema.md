# Figma Audit Reference: Style Capture, Normalization & JSON Schema

## Step 4 — Style Capture Snippet & Field Mapping

### 4a — Capture + normalize

```js
const text = await figma.getLocalTextStylesAsync();
const fx = await figma.getLocalEffectStylesAsync();
const toHex = /* same helper as Step 3 */;

// Map Figma font-style names to numeric font-weight per CSS spec.
// Order matters: check more-specific names before generic ones (e.g. "extrabold" before "bold").
const weightFromStyle = (style) => {
  const s = String(style || '').toLowerCase();
  if (s.includes('thin')) return 100;
  if (s.includes('extralight') || s.includes('extra light') || s.includes('ultralight')) return 200;
  if (s.includes('light')) return 300;
  if (s.includes('medium')) return 500;
  if (s.includes('semibold') || s.includes('semi bold') || s.includes('demibold')) return 600;
  if (s.includes('extrabold') || s.includes('extra bold') || s.includes('ultrabold')) return 800;
  if (s.includes('black') || s.includes('heavy')) return 900;
  if (s.includes('bold')) return 700;
  return 400; // default for "Regular" / "Normal" / unknown
};

// Figma lineHeight: { value, unit: 'PIXELS' } | { value, unit: 'PERCENT' } | { unit: 'AUTO' }.
// Emit as a px string; AUTO becomes null (apply will treat as "unspecified").
const lineHeightToPx = (lh, fontSize) => {
  if (!lh || lh.unit === 'AUTO') return null;
  if (lh.unit === 'PIXELS') return `${lh.value}px`;
  if (lh.unit === 'PERCENT') return `${(lh.value / 100) * fontSize}px`;
  return null;
};

const figma_styles = [
  ...text.map(s => ({
    name: s.name,
    type: 'text',
    decomposed: {
      size: `${s.fontSize}px`,
      lineHeight: lineHeightToPx(s.lineHeight, s.fontSize),
      fontWeight: weightFromStyle(s.fontName.style),
      fontFamily: s.fontName.family,
    },
  })),
  ...fx.map(s => ({
    name: s.name,
    type: 'shadow',
    effects: s.effects.map(e => ({
      kind: e.type === 'DROP_SHADOW' ? 'drop-shadow'
          : e.type === 'INNER_SHADOW' ? 'inner-shadow'
          : e.type.toLowerCase().replace(/_/g, '-'),
      offset_x: e.offset ? `${e.offset.x}px` : '0px',
      offset_y: e.offset ? `${e.offset.y}px` : '0px',
      blur: `${e.radius ?? 0}px`,
      spread: `${e.spread ?? 0}px`,
      color: e.color ? toHex({ r: e.color.r, g: e.color.g, b: e.color.b, a: e.color.a }) : null,
    })),
  })),
];

return figma_styles;
```

### 4b — Field mapping (capture → emission)

| Figma-API field | Emitted as |
|---|---|
| `s.fontSize` (number, px) | `decomposed.size: "{N}px"` |
| `s.lineHeight` (object) | `decomposed.lineHeight: "{N}px"` (or `null` if AUTO) |
| `s.fontName.style` (e.g., `"Bold"`) | `decomposed.fontWeight: 700` (numeric, via `weightFromStyle`) |
| `s.fontName.family` | `decomposed.fontFamily` |
| `s.letterSpacing` | **dropped** — apply skills do not use it; reintroduce only when a consumer needs it |
| `e.type === 'DROP_SHADOW'` | `kind: "drop-shadow"` |
| `e.offset.x` / `e.offset.y` (numbers) | `offset_x: "{N}px"` / `offset_y: "{N}px"` |
| `e.radius` (Gaussian blur) | `blur: "{N}px"` (renamed — `radius` is misleading) |
| `e.spread` | `spread: "{N}px"` |
| `e.color: { r, g, b, a }` | `color: "#RRGGBB[AA]"` (via `toHex`) |

Heads-up on shadow effects: `radius` on a Figma drop-shadow is the **Gaussian blur radius** (CSS `blur-radius`), not `border-radius`. The capture template renames it to `blur` precisely to avoid this confusion downstream.



### 5e — Normalization rules

Normalize before comparing. None of these are drift:

**Color (hex) values:**
- Case: `#FF0000` vs `#ff0000`
- Shorthand expansion: `#f00` vs `#ff0000`
- Fully-opaque alpha equivalence: `#ff0000ff` vs `#ff0000`
- Surrounding whitespace

For colors, lowercase and expand both sides to canonical 6- or 8-digit hex before comparing.

**Numeric (FLOAT) values — spacing, radius, sizes:**
- Figma returns FLOAT primitives in **pixels** (raw numbers). When the CSS uses `rem` (16px base unless the project overrides `font-size` on `:root` / `html`), convert Figma px → rem before comparing: `8 (Figma)` ↔ `0.5rem (CSS)` is a match. When the CSS uses `px`, compare as-is.
- Unit tolerance: `0` vs `0px` vs `0rem` are equivalent.
- Trailing-zero / format equivalence: `0.5rem` vs `0.50rem`, `1rem` vs `1.0rem` — match.
- If the project base font-size is non-default (e.g., 10px), use that base for the conversion. Detect by parsing `:root { font-size: ... }` if present.

For other numeric uses (opacity, line-height multipliers, etc.), normalize to a common decimal representation with no trailing zeros.

**String values (Figma `STRING` resolved type — typically font-family or arbitrary text tokens):**
- Trim leading/trailing whitespace.
- Collapse internal multiple spaces to a single space.
- Preserve case (CSS string values like `Inter, sans-serif` are case-significant for font-family lookup).
- Quote-style differences are NOT drift: `'Inter'`, `"Inter"`, and unquoted `Inter` are equivalent for single-token strings. For multi-token / comma-separated lists, normalize each segment the same way.

**Boolean values (Figma `BOOLEAN` resolved type):**
- BOOLEAN variables have no direct CSS equivalent. Do not emit them as `drift` or `missing-in-css`. Instead, append to a top-level `unsupported_for_css[]` array with `{ figma_source, type: "boolean", value: <true|false>, reason: "no CSS variable equivalent" }`. Apply skills surface these in section 4 of the final report for visibility, but never attempt Edits.

## Step 6 — Emit JSON

Write `docs/figma-audit.json`. Audit overwrites it on every run.

Schema:

```json
{
  "schema_version": "2.0",
  "generated_at": "2026-05-06T22:30:00Z",
  "figma": { "fileKey": "abc...", "branchKey": null, "nodeId": "0:1" },
  "css_file": "next-frontend/app/globals.css",
  "css_file_sha": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "css_blocks": [
    { "selector": ":root", "condition": null },
    { "selector": ".dark", "condition": null },
    { "selector": "@theme inline", "condition": null },
    { "selector": ":root", "condition": "@media (prefers-color-scheme: dark)" }
  ],
  "_meta": {
    "mode_to_block": {
      "Light": { "selector": ":root", "condition": null },
      "Dark":  { "selector": ".dark", "condition": null }
    },
    "unmapped_modes": [],
    "styles_capture": "local-only"
  },
  "items": [
    {
      "kind": "drift",
      "token": "--ring",
      "selector": ".dark",
      "condition": null,
      "from": { "$value": "#fb3748" },
      "to":   { "$value": "#e8203a" },
      "figma_collection": "Theme",
      "figma_source": "Theme/ring#Dark"
    },
    {
      "kind": "drift",
      "token": "--background",
      "selector": ":root",
      "condition": "@media (prefers-color-scheme: dark)",
      "from": { "$value": "#0a0a0a" },
      "to":   { "$value": "#0f0f0f" },
      "figma_collection": "Theme",
      "figma_source": "Theme/background#Dark"
    },
    {
      "kind": "missing-in-css",
      "token": "--warning",
      "figma_collection": "Theme",
      "values_by_block": [
        { "selector": ".dark", "condition": null, "$value": "#d97706" }
      ],
      "figma_source": "Theme/warning"
    },
    {
      "kind": "missing-in-css",
      "token": "--color-neutral-100",
      "figma_collection": "Primitives",
      "value": { "$value": "#f7f7f7" },
      "figma_source": "Primitives/colors/neutral/100"
    }
  ],
  "figma_styles": [
    {
      "name": "Heading/Display",
      "type": "text",
      "decomposed": { "size": "40px", "lineHeight": "48px", "fontWeight": 700, "fontFamily": "Inter" }
    },
    {
      "name": "Drop Shadow / Card",
      "type": "shadow",
      "effects": [
        { "kind": "drop-shadow", "offset_x": "0px", "offset_y": "10px", "blur": "15px", "spread": "0px", "color": "#00000010" }
      ]
    }
  ],
  "external_library_aliases": [
    {
      "figma_source": "Theme/ring",
      "reason": "MISSING_TARGET",
      "affected_blocks": [{ "selector": ".dark", "condition": null }]
    },
    { "figma_source": "Primitives/colors/red/500", "reason": "MISSING_TARGET", "affected_blocks": [] }
  ],
  "cross_block_warnings": [
    {
      "kind": "unexpected-block",
      "token": "--ring",
      "unexpected_block": { "selector": "@theme inline", "condition": null },
      "expected_blocks": [
        { "selector": ":root", "condition": null },
        { "selector": ".dark", "condition": null }
      ],
      "reason": "multi-mode token defined in non-mapped block"
    },
    {
      "kind": "multiple-declarations",
      "token": "--foo",
      "selector": ":root",
      "condition": null,
      "values_seen": ["#abc", "#def"],
      "reason": "same (selector, condition, var) declared with different values across multiple blocks"
    },
    {
      "kind": "conditional-block",
      "token": "--ring",
      "selector": ":root",
      "condition": "@media (prefers-reduced-motion: reduce)",
      "reason": "variable declared inside conditional at-rule; excluded from drift comparison"
    }
  ],
  "unsupported_for_css": [
    { "figma_source": "FeatureFlags/use-new-shadow", "type": "boolean", "value": true, "reason": "no CSS variable equivalent" }
  ]
}
```

**Schema notes:**

- Token values use the shape `{ "$value": <string> }` (or `$value` directly on entries of `values_by_block[]`). The `$value` key is borrowed from the W3C DTCG vocabulary for familiarity, but no other DTCG features (`$type`, aliases, etc.) are emitted. Apply skills read `item.from.$value`, `item.to.$value`, `item.value.$value`, and `block.$value` (per entry of `values_by_block[]`) directly.
- **A "block" is a `{ selector, condition }` pair**, where `condition` is `null` for top-level blocks and a canonical at-rule string (per Step 2) for promoted conditional blocks. This shape appears uniformly across `css_blocks[]`, drift items (`selector` + `condition` fields), `values_by_block[]` entries, `_meta.mode_to_block` values, `external_library_aliases[].affected_blocks[]`, and `cross_block_warnings[].unexpected_block` / `expected_blocks[]`.
- `figma_collection` is included on **every** `items[]` entry (drift and missing-in-css alike) for traceability and final-report grouping. It is informational; apply skills route by item shape (`value` vs `values_by_block`), not by this field.
- `figma_source` follows a single canonical format: `"<collection>/<full-figma-name>[#<mode>]"`. The trailing `#<mode>` segment is present only on `drift` items (where the diff is per-mode). Missing-in-css items omit it because the item already groups all modes via `values_by_block` or is single-mode via `value`.
- `_meta.mode_to_block` is informational: it preserves the audit's mapping decision for human inspection / future tooling. Apply skills do not consume it; block information needed for editing lives directly on items (`selector` + `condition` for drift, entries of `values_by_block[]` for theme missing).
- `_meta.unmapped_modes[]` lists Figma mode names the user explicitly chose not to map to any block during Step 5c (e.g., `["Print"]`). Always present as `[]` when all modes are mapped. Apply Step 7 sec 4 surfaces the list when non-empty so users can confirm intentional vs accidental exclusion.
- **Optional top-level arrays are always present, never omitted.** `external_library_aliases[]`, `cross_block_warnings[]`, `unsupported_for_css[]`, and `_meta.unmapped_modes[]` MUST be emitted as `[]` when empty. This lets apply skills read them via `audit.X` without null-guards or `?? []` fallbacks. Same rule for `figma_styles[]` and `items[]` (they are empty `[]` when there is nothing to report, never absent).
- **`cross_block_warnings[]` entries are discriminated by `kind`.** Three kinds exist: `"multiple-declarations"` (same `(selector, condition, var)` declared with different values across multiple blocks of the same `(selector, condition)` — has `selector` + `condition` + `values_seen[]` fields), `"unexpected-block"` (a token defined in a block outside its expected blocks per Step 5d.ii — has `unexpected_block` (a `{selector, condition}` object) + `expected_blocks[]` (array of `{selector, condition}` objects) fields), and `"conditional-block"` (a token declared inside a non-allowlisted conditional at-rule like `@media` / `@supports` / `@container` — has `selector` + `condition` fields; excluded from drift comparison per Step 2). Apply skills should branch on `kind` when formatting; consumers must not assume one shape.

