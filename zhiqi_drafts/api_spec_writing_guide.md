# API Specification Writing Guide

This guide defines how to write API specifications in this repository so that both humans and LLMs produce consistent, implementation-ready contracts.

It mirrors the structure and rigor of the current reference spec at [docs/api_spec_repo/en_US.md](../en_US.md).

---

## 0. Intent and Audience

**Audience:**

- Product owners and domain experts
- Backend engineers
- Frontend engineers
- QA engineers
- LLM agents drafting or revising specs

**Primary goal:**
Write specs that are unambiguous, testable, and stable across backend, frontend, and QA.

**Secondary goal:**
Keep the document readable enough for non-engineers while preserving machine-checkable precision.

---

## 1. Authoring Principles

Use the following rules in every API spec:

1. Backend owns computed state.

- If a field is derived (for example status), define it as backend-owned and non-writable by client.

2. Method semantics must be explicit.

- `POST` creates (or appends) a new resource.
- `PUT` replaces/updates idempotently.
- `DELETE` removes.

3. Define invariants and edge cases near the affected endpoint.

- Example: draft allowed before prerequisites complete.
- Example: finalize blocked until minimum conditions are met.

4. Separate protocol concerns from business rules.

- HTTP and envelope format belong in common sections.
- Domain behavior belongs in status machine and endpoint sections.

5. Prefer explicitness over brevity.

- Name all required headers.
- List exact validation errors.
- Include conflict scenarios (`409`) where state transitions can fail.

6. Keep response contracts frontend-friendly.

- Return fully updated resource objects after mutating operations when possible.
- Avoid requiring redundant follow-up `GET` calls unless necessary.

---

## 2. Canonical Document Structure

Follow this order unless there is a strong reason not to:

1. Design Principles
2. Common Interfaces

- Base response envelope
- Error object
- Error type reference

3. HTTP Status Codes
4. Authentication
5. State Machine (if stateful domain)
6. Endpoints
7. Domain Formulas / Reference Calculations

If a section is not applicable, explicitly state "Out of scope" rather than deleting the section silently.

---

## 3. Required Common Interfaces

### 3.1 Base Response Envelope

All responses must use one envelope shape:

```json
{
  "status": "success | error",
  "code": "integer (HTTP status code)",
  "timestamp": "string (ISO 8601)",
  "apiVersion": "string (e.g., 'v1')",
  "data": "object | array | null",
  "error": "object | null"
}
```

Rules:

- Success response: `data != null` (except where null is intentional), `error = null`.
- Error response: `data = null`, `error != null`.
- Always include `apiVersion`.

### 3.2 Error Object

```json
{
  "type": "string",
  "message": "string",
  "details": [
    {
      "field": "string | null",
      "message": "string"
    }
  ]
}
```

Rules:

- `details` is required (can be empty).
- Use dot-path for `field` when possible (for example `reviewers[1].email`).

### 3.3 Error Type Registry

Define a project-level table and reuse it consistently. At minimum, include:

- `VALIDATION_ERROR`
- `AUTHENTICATION_ERROR`
- `AUTHORIZATION_ERROR`
- `NOT_FOUND`
- `CONFLICT`
- `TIMEOUT_ERROR`
- `RATE_LIMIT_ERROR`
- `INTERNAL_SERVER_ERROR`

Add domain-specific error types only when truly needed.

---

## 4. HTTP Status and Error Mapping Rules

For each endpoint, specify both:

- HTTP code
- Error type and precise trigger condition

Minimum mapping discipline:

- `400`: malformed/invalid request, validation failure
- `401`: missing/invalid/expired auth
- `403`: authenticated but not permitted
- `404`: resource absent
- `409`: state conflict or lifecycle violation
- `429`: throttling
- `500`: unexpected backend failure

Do not use `500` for expected business constraints.

---

## 5. Authentication Section Rules

Authentication section must state:

1. Which endpoints are public (if any).
2. Required authenticated headers.
3. Token/session lifecycle assumptions (expiry, refresh, re-auth trigger).
4. Behavior on expiry during unsaved work (if relevant).

If auth is handled externally (for example SSO vendor), write that explicitly and scope frontend responsibilities.

---

## 6. State Machine Authoring Rules

When the domain has lifecycle states, include:

1. State value table.

- Internal value
- Display label
- Meaning

2. Transition conditions.

- Use readable flow notation.
- For each transition, provide trigger and mandatory preconditions.

3. Business rules.

- Draft-save rules
- Finalization constraints
- Immutability after completion
- Regression edge cases

4. Conflict policy.

- Explicitly state which invalid transitions return `409 CONFLICT`.

---

## 7. Endpoint Writing Template

Use this exact endpoint section pattern.

### `METHOD /path` - Purpose

**Path Parameter:**

| Parameter | Type        | Description |
| --------- | ----------- | ----------- |
| `id`      | UUID string | Resource id |

**Query Parameters:**

- List each field, type, optionality, defaults.

**Request Body:**

```json
{
  "field": "type"
}
```

**Validation Rules:**

- Rule 1
- Rule 2

**Response (`2xx`):**

```json
{
  "status": "success",
  "code": 200,
  "timestamp": "string (ISO 8601)",
  "apiVersion": "v1",
  "data": {},
  "error": null
}
```

**Errors:**

| Code | ErrorType               | Condition |
| ---- | ----------------------- | --------- |
| 400  | `VALIDATION_ERROR`      | ...       |
| 401  | `AUTHENTICATION_ERROR`  | ...       |
| 403  | `AUTHORIZATION_ERROR`   | ...       |
| 404  | `NOT_FOUND`             | ...       |
| 409  | `CONFLICT`              | ...       |
| 500  | `INTERNAL_SERVER_ERROR` | ...       |

**Implementation Notes:**

- Add frontend state update guidance where useful.
- Clarify server-generated fields.
- Clarify idempotency behavior.

---

## 8. Contract Precision Conventions

Use these conventions consistently:

1. Time format

- Always specify `ISO 8601`.
- If timezone matters, state requirement for UTC offset.

2. Nullability

- Use `type | null` explicitly.
- Explain semantic meaning of null.

3. Arrays

- State whether empty array is allowed.
- State minimum item constraints where needed.

4. Enumerations

- Inline exact allowed values (for example `待推薦 | 待初審 | ...`).

5. Ownership

- Mark server-generated fields (for example created/update timestamps).
- Mark read-only fields.

6. Cross-field constraints

- Express dependencies (for example if `isFinalized = true`, then `score` and `remarks` must be non-null).

---

## 9. Versioning and Change Management

Required practices:

1. Include version in envelope (`apiVersion`).
2. Define major/minor policy at top of document.
3. For breaking changes:

- Increment major version.
- Add migration note.
- List affected endpoints.

4. Keep an update note section for date-stamped behavior changes.

---

## 10. LLM Instruction Block (Reusable)

Use this block as prompt context when asking an LLM to draft or revise an API spec.

```text
You are writing an API specification for engineering delivery.
Follow these constraints strictly:
1) Use the canonical section order: Design Principles, Common Interfaces, HTTP Status Codes, Authentication, State Machine (if applicable), Endpoints, Reference Calculations.
2) Use one unified response envelope with status/code/timestamp/apiVersion/data/error.
3) For every endpoint, include path/query params, request body, validation rules, success response, and an explicit error mapping table (Code + ErrorType + Condition).
4) Specify nullability, enum values, and server-vs-client ownership for each sensitive field.
5) If the domain has lifecycle status, provide a full transition model and `409 CONFLICT` conditions.
6) Prefer frontend-efficient mutation responses (return updated resource object where feasible).
7) Use precise, testable language; avoid vague terms like "normally" or "should usually".
8) Preserve backward compatibility assumptions and state versioning implications.
```

---

## 11. Human Review Checklist

Before finalizing a spec, verify all items:

1. Every endpoint has complete request/response/error definitions.
2. Every validation rule has corresponding error behavior.
3. Every lifecycle transition has clear trigger and conflict rule.
4. No contradictions between endpoint sections and state machine.
5. `POST`/`PUT` semantics are consistent and intentional.
6. Nullability and required fields are explicit.
7. Timestamps and IDs have defined formats.
8. Auth behavior and failure modes are fully stated.
9. Versioning impact is clear for changed behavior.
10. Frontend can implement without guessing.

---

## 12. Anti-Patterns to Avoid

Do not do the following:

1. Omit error tables and rely on "standard errors apply".
2. Hide crucial constraints only in prose paragraphs.
3. Use inconsistent names for the same concept across endpoints.
4. Mix computed backend fields with writable client fields without ownership note.
5. Return partial mutation results that force unnecessary read-after-write requests.
6. Define states without transition conditions.

---

## 13. Minimal Skeleton (Copy/Paste)

```markdown
# <Project/Domain> API Specification

**Scope:** ...
**Versioning:** ...

## 1. Design Principles

...

## 2. Common Interfaces

### 2.1 Base Response Envelope

### 2.2 Error Object

### 2.3 ErrorType Reference

## 3. HTTP Status Codes

...

## 4. Authentication

...

## 5. State Machine

### 5.1 Status Values

### 5.2 Transition Conditions

### 5.3 Business Rules

## 6. Endpoints

### 6.1 `GET /...`

### 6.2 `POST /...`

### 6.3 `PUT /...`

...

## 7. Reference Calculations

...
```

---

## 14. Repository-Specific Notes for This Project

1. Maintain separate track-aware paths when backend domains are split (for example parallel route families for different programs).
2. Keep status labels and lifecycle semantics aligned across list and detail endpoints.
3. Ensure mutation endpoints that affect reviewer recommendations and final review return objects usable for direct frontend state updates.
