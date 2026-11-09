# Plan Phase — Output Structure Template

## Output structure

The plan follows a fixed structure. Each phase generates a single markdown file.

> **Reading the template below:** text rendered as `_(italic in parentheses)_` is author guidance about *when* to include the surrounding section or subsection. Do **not** copy these parentheticals into the generated phase document — apply them as conditions and include the block only when the condition holds.

```markdown
# Phase NN — [Phase Name]

## Objective

One sentence summarizing what this phase delivers.

---

## Step Implementations

### SI-NN.1 — [SI name]

**Description:** What this step implementation implements, in one or two sentences.

**Technical actions:**

- Concrete implementation action
- Concrete implementation action
- Concrete implementation action

**Tests:** _(omit if the SI creates no testable artifacts)_

| File | Layer | Verifies |
|------|-------|----------|
| file.spec.ts | Unit/Integration/E2E | what this file tests |

**Dependencies:** SI-NN.X (comma-separated if multiple, e.g., `SI-02.1, SI-02.3`) or None

**Acceptance criteria:**

- Verifiable criterion specific to this step implementation
- Verifiable criterion specific to this step implementation

---

(repeat for each step implementation)

## Technical Specifications

_(include only the subsections that apply to the phase — see applicability checklist below)_

### Data Model

#### [EntityName]

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uuid | PK, generated | |
| field | type | constraints | |

**Relations:** EntityName → OtherEntity (many-to-one)
**Indexes:** (field_a, field_b) — unique

---

### API Contracts

#### METHOD /resource/path (SI-NN.X)

**Request headers:**
- Authorization: Bearer <access_token> (if authenticated)
- Content-Type: application/json

**Request body:**
- field: type, required/optional — constraints

**Response 2XX:** _(choose based on REST Conventions: 200 for data, 201 for creation, 204 for no content)_
- field: type _(omit entire section for 204 — no body)_

**Response headers:** _(if applicable)_
- Location: /resource/:id _(for 201 when GET endpoint exists)_
- Header-Name: description

**Error responses:**
- 4XX ERROR_CODE: when this error occurs (for domain errors with codes from the Error Catalog)
- 400 validation error: when the request body fails schema validation (no domain code)

#### Validation Rules — [EntityName or endpoint] _(optional — only when validation rules are extensive)_

| Field | Rule | Error message |
|-------|------|---------------|
| field | rule description | message returned on violation |

---

### Authorization Matrix

| Endpoint | Public | Authenticated | Role |
|----------|--------|---------------|------|
| GET /resource | ✓ | | |
| POST /resource | | ✓ | |
| DELETE /resource/:id | | | OWNER or ADMIN |

---

### Error Catalog

**Error response format:** _(include only in the first phase that introduces HTTP endpoints in this subproject — later phases in the same subproject inherit this shape and do not redefine it)_
```
{ statusCode, error, message }
```
_The `error` field carries the domain error code from the Error Catalog (e.g., `"EMAIL_ALREADY_EXISTS"`). Adjust shape per subproject conventions._

| Code | HTTP | Message | Trigger |
|------|------|---------|---------|
| ERROR_CODE | 4XX | Human-readable message | Which operation + which branch triggers it |

---

### Events/Messages

| Event | Payload | Publisher | Consumer | Delivery |
|-------|---------|-----------|----------|----------|
| event.name | { field: type } | ServiceName | WorkerName | fire-and-forget / ack-required |

---

## Dependency Map

(text tree showing implementation order — one node per SI, children are SIs that depend on the parent)

```
SI-NN.1 (no deps)
├── SI-NN.2
│   └── SI-NN.4
└── SI-NN.3
    └── SI-NN.5
```

## Deliverables

- [ ] Verifiable deliverable
- [ ] Verifiable deliverable
- [ ] All SI tests pass (`<test command for subproject>`)
- [ ] E2E tests pass (`<e2e test command, if applicable>`)
- [ ] Type/compilation check passes (`<type-check command for subproject>`)
- [ ] Project builds successfully (`<build command for subproject>`)
```

