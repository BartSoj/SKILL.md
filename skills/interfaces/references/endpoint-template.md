# Endpoint and Event Block Templates

Every entry in § 6 Endpoints and § 7 Events of INTERFACES.md follows the exact
block structure below. The agent must apply these templates verbatim — field
order, heading level, bullet labels, and JSON fence style all matter because
downstream agents parse these blocks to derive request/response types, client
stubs, and test fixtures.

---

## Endpoint Block

```markdown
### `EP-{stable-id}` — `{METHOD} {/path}`

- **Boundary:** `{boundary name from § 1}`
- **Purpose:** {one sentence in domain terms}
- **Use cases:** `UC-{NN}`, `UC-{NN}`
- **Auth:** `{scheme from § 3}` — `{scope or role required, or "any authenticated user", or "none"}`
- **Idempotency:** `yes (via Idempotency-Key header)` | `yes (naturally idempotent method)` | `no`
- **Request:**
  - **Path params:**
    | Name | Type | Constraint | Required |
    |------|------|-----------|----------|
    | `{name}` | `{wire type}` | `{constraint or —}` | `yes` / `no` |
  - **Query params:**
    | Name | Type | Constraint | Required | Default |
    |------|------|-----------|----------|---------|
    | `{name}` | `{wire type}` | `{constraint}` | `yes` / `no` | `{value or —}` |
  - **Headers (beyond auth & versioning):**
    | Name | Required | Notes |
    |------|----------|-------|
    | `{Header-Name}` | `yes` / `no` | `{semantics}` |
  - **Body:**
    ```json
    {
      "fieldName": "type — required/optional. One-line description. Constraint.",
      "nestedField": {
        "innerField": "type — required/optional. Description."
      }
    }
    ```
    (If no body: "No request body.")
- **Response 2xx (`{200 | 201 | 204}`):**
  ```json
  {
    "fieldName": "type. One-line description.",
    "collection": [
      {
        "itemField": "type. Description."
      }
    ]
  }
  ```
  (If `204 No Content`: "No response body.")
- **Response errors:** `ERR_CODE_ONE` (`{http status}`), `ERR_CODE_TWO` (`{http status}`), `ERR_CODE_THREE` (`{http status}`) — full definitions in ERRORS.md.
- **Events emitted:** `EVT-{name-one}`, `EVT-{name-two}` (from § 7) — or `(none)`.
- **Transformation notes:** {If the handler reshapes data between boundaries — e.g., receives snake_case from an internal service and emits camelCase to the client — describe both shapes and the transformation. Omit this bullet entirely if there is no transformation.}
- **Rate limits:** `{N requests per minute per {scope}}` — or `(none)`.
```

**Field rules:**

- `EP-{stable-id}` is kebab-case and names the action, not the resource path (`EP-push-create`, `EP-repo-read`, `EP-user-list`). Assigned once; never renumbered.
- `{METHOD}` is uppercase (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`). `{/path}` uses the boundary's versioning convention (`/v1/repos/{repoId}/push`).
- Wire types are literal JSON types: `string`, `number`, `integer`, `boolean`, `array<T>`, `object`, `null`. Language-native type names (`i64`, `uint32`, `DateTime<Utc>`) are forbidden — they are implementation details.
- Every field in every JSON block carries `type — required/optional. Description.` — all three parts present, separated by ` — ` and `. `.
- Error codes appear as ID citations only. Never inline the message, never inline the status-to-meaning mapping. Those live in ERRORS.md.
- If the endpoint emits no events, write `(none)`. If the project has no events at all, `§ 7` itself states so and this bullet always reads `(none)`.

---

## Event Block

```markdown
### `EVT-{name-in-kebab-case-past-tense}`

- **Kind:** `domain event (in-process)` | `integration event (cross-service)` | `webhook (outbound)`
- **Producer:** `{component name from ARCHITECTURE.md}`
- **Consumers:** `{component}`, `{component}` — or `(fan-out; any subscriber)`
- **Delivery guarantee:** `at-most-once` | `at-least-once` | `exactly-once (via idempotency key {field})`
- **Ordering:** `none` | `per-partition-key {key-field}`
- **Related use cases:** `UC-{NN}`, `UC-{NN}`
- **Payload:**
  ```json
  {
    "fieldName": "type — required/optional. One-line description.",
    "nested": {
      "innerField": "type — required/optional. Description."
    }
  }
  ```
- **Headers / metadata (beyond envelope):**
  | Name | Required | Notes |
  |------|----------|-------|
  | `event-id` | `yes` | stable identifier for deduplication |
  | `occurred-at` | `yes` | ISO 8601 UTC |
  | `schema-version` | `yes` | integer, increments on breaking change |
  | `{custom-header}` | `yes` / `no` | `{semantics}` |
- **Version:** `{N}` — current schema version.
- **Evolution rules:** `{additive only}` | `{breaking changes require new EVT-{name-v2}}` | `{see § 8 Evolution Policy, class {X}}`
```

**Field rules:**

- `EVT-{name}` is kebab-case, past tense, names a state change (`EVT-push-accepted`, `EVT-order-placed`, `EVT-subscription-renewed`). Commands (`EVT-place-order`) are forbidden — those belong to BEHAVIOR.md or endpoint request bodies.
- `Producer` is a single component name. If more than one component can emit the event, split into separate events or introduce a discriminator field and note the producer as "any of {list}".
- `Delivery guarantee` is one of the three literal strings. `exactly-once` always names the idempotency key field in parentheses.
- `Ordering: per-partition-key {key-field}` names the exact payload field used for partitioning.
- `Payload` follows the same wire-type rules as endpoint bodies. Payload fields name *business* information, not transport metadata (transport metadata lives under `Headers / metadata`).
- `schema-version` in headers is the *event* schema version, not the API version. It increments when the payload shape changes.

---

## Worked Example — Endpoint

```markdown
### `EP-push-create` — `POST /v1/repos/{repoId}/push`

- **Boundary:** `Server API ↔ Client`
- **Purpose:** Submit a push to a repository; on success the server accepts the pushed objects and updates the branch ref.
- **Use cases:** `UC-04`, `UC-11`
- **Auth:** `Bearer JWT` — `repo:write` scope on `{repoId}`
- **Idempotency:** `yes (via Idempotency-Key header)`
- **Request:**
  - **Path params:**
    | Name | Type | Constraint | Required |
    |------|------|-----------|----------|
    | `repoId` | `string` | `^repo_[a-z0-9]{8,}$` | `yes` |
  - **Headers:**
    | Name | Required | Notes |
    |------|----------|-------|
    | `Idempotency-Key` | `yes` | ULID, 15-min retention |
  - **Body:**
    ```json
    {
      "branch": "string — required. Target branch ref (e.g., 'main'). Max 250 chars.",
      "parentSha": "string — required. 40-char hex; the tip the client believes the branch is at.",
      "objects": "array<object> — required. Pack objects; each { sha: string, kind: string, bytes: integer }."
    }
    ```
- **Response 2xx (`201`):**
  ```json
  {
    "pushId": "string. ULID of the accepted push.",
    "newSha": "string. 40-char hex of the new branch tip.",
    "acceptedAt": "string. ISO 8601 UTC timestamp."
  }
  ```
- **Response errors:** `REPO_NOT_FOUND` (`404`), `PUSH_CONFLICT` (`409`), `INVALID_PACK` (`400`), `AUTH_REQUIRED` (`401`), `FORBIDDEN` (`403`) — full definitions in ERRORS.md.
- **Events emitted:** `EVT-push-accepted`
- **Rate limits:** `60 requests per minute per user`.
```

## Worked Example — Event

```markdown
### `EVT-push-accepted`

- **Kind:** `integration event (cross-service)`
- **Producer:** `Access Service`
- **Consumers:** `Storage Service`, `Audit Service`
- **Delivery guarantee:** `at-least-once`
- **Ordering:** `per-partition-key repoId`
- **Related use cases:** `UC-04`
- **Payload:**
  ```json
  {
    "pushId": "string — required. ULID of the accepted push.",
    "repoId": "string — required. Repository identifier.",
    "branch": "string — required. Target branch ref.",
    "newSha": "string — required. New branch tip, 40-char hex.",
    "actorId": "string — required. User who pushed."
  }
  ```
- **Headers / metadata:**
  | Name | Required | Notes |
  |------|----------|-------|
  | `event-id` | `yes` | ULID, distinct from `pushId` |
  | `occurred-at` | `yes` | ISO 8601 UTC |
  | `schema-version` | `yes` | currently `1` |
- **Version:** `1`
- **Evolution rules:** additive only within v1; breaking changes introduce `EVT-push-accepted-v2`.
```
