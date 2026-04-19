---
name: SECURITY.md
description: Model the system's threats and controls in one consolidated artifact — assets, hostile actors, trust boundaries, STRIDE threat catalogue, mitigations tied to implementation, authentication and authorization, secret management, input validation, abuse prevention, supply-chain security, incident response, and residual risks. Use when asked to model security threats, write the threat model and controls, design auth and abuse prevention, or produce a SECURITY.md.
---

# Task: Generate SECURITY.md — Threat Model & Controls

## Objective

Produce a SECURITY.md that serves as the single source of truth for the system's threat model and the controls that address it: assets worth protecting, hostile actor classes with capability assumptions, trust boundaries inherited from INTERFACES, a data-flow diagram flagging every boundary crossing, a STRIDE-driven threat catalogue under stable `THREAT-NN` IDs, mitigations under stable `MIT-NN` IDs tied to concrete implementation locations, authentication and authorization models (role inventory, permission model, enforcement points, use-case-to-permission map), secret management, input validation and output encoding per trust boundary, abuse prevention, supply-chain security, incident response, and an explicit residual-risk register. An agent reading this document alone can — for any endpoint, saga, or data store — name the threats it faces, the mitigations that cover them, the enforcement point for each authorization check, and the residual risks the business has accepted, without opening another file.

SECURITY.md consolidates what older guidance split into THREAT_MODEL.md and a controls catalogue, because the two are always read together: a threat without a mitigation is noise, a mitigation without the threat it addresses is incomplete. The defining discipline — and the commonest failure — is **every `THREAT-NN` has at least one `MIT-NN` addressing it or an explicit residual-risk acceptance, and every `MIT-NN` names a concrete implementation location**. Orphan threats are modelling aspirations; orphan mitigations are controls with no owner.

---

## Inputs

1. **ARCHITECTURE.md** (required) — § 2 Components, cross-component flows, and deployment shape frame assets, trust boundaries, and the DFD; `ADR-NN` on zero-trust, segmentation, or secret-storage informs §§ 7–9.
2. **DOMAIN.md** (required) — § 1 Glossary supplies vocabulary; §§ 4–6 supply asset candidates; § 9 Invariants Index supplies invariants whose violation is itself a security event.
3. **INTERFACES.md** (required) — § 1 Boundaries is authoritative for § 3 Trust Boundaries and DFD crossings; § 3 Authentication, § 4 Idempotency, § 6 Endpoints supply the auth scheme, replay semantics, and endpoint inventory this document annotates.
4. **Surface IAs** (required — one per existing surface: `WEB_IA.md`, `CLI_IA.md`, `MOBILE_IA.md`, `TUI_IA.md`, `VOICE_IA.md`) — auth UX, form-validation contract, error visibility policy; § 7 and § 10 cite IA flows.
5. **USE_CASES.md** (required) — § 1 Actors map to § 2.1 Authorized Actors; § 2 Use Cases with stable `UC-NN` IDs drive the § 8 use-case-to-permission map.
6. **ERRORS.md** (optional — required if produced) — § 3 Error Code Registry supplies security-signal codes (`AUTHN_FAILED`, `AUTHZ_DENIED`, `RATE_LIMITED`) cited in §§ 5, 8, 11.
7. **DATA.md** (optional — if produced) — § 8 sensitive-column inventory informs § 1 and § 10; retention policies inform § 14.
8. **OPERATIONS.md** (optional) — § 9 storage locations and § 13 paging rules cite it; absence surfaces as open questions.
9. **QUALITY.md** (optional) — § 6 alerts cited from § 13; security-specific detection rules live here, not in QUALITY.
10. **Existing SECURITY.md** (auto-discovered — only if refining) — read fully. Every assigned `THREAT-NN` and `MIT-NN` is permanent; retired entries retain their row with `(retired — superseded by {new-id})`.

Read all required inputs end-to-end. Truncated reads cause two specific failure modes: trust boundaries missing from § 3 (INTERFACES § 1 not fully read) and authorization rules missing `UC-NN` references (USE_CASES not fully read).

---

## Workflow

Construction proceeds in eight phases: assets and actors, trust boundaries and DFD, STRIDE enumeration, mitigations, auth models, secrets and validation, abuse and supply chain and IR, residual risks and validation. Phases are sequential — later phases cite IDs from earlier phases — but revisit earlier phases if a later one reveals a missing asset, an un-mitigated threat, or an elided boundary.

Before Phase 3, read `references/stride-checklist.md` for the per-element × per-STRIDE-category prompts applied to every DFD element. The checklist owns systematic enumeration; the workflow below describes what to do with its output.

### Phase 1: Assets & Actors

§ 1 Assets enumerates what the system protects. Derive from DOMAIN §§ 4–6 (entities and aggregates representing protected state), DATA § 8 when present (sensitive columns), and INTERFACES § 6 (anything an endpoint reads or writes that matters to the business). Each entry fills the Output Format template: name (DOMAIN canonical form), CIA classification each rated low/medium/high, location citing ARCHITECTURE § 2 components, owning bounded context, exposure.

§ 2 Actors has two groups. **Authorized actors** are a one-to-one map of USE_CASES § 1 — never invent new ones here, never drop one. **Unauthorized / hostile classes** cover archetypes relevant to the system: anonymous attacker, authenticated-but-unauthorized user, compromised dependency, malicious insider, supply-chain attacker, nation-state where relevant. Capability assumptions (network-only? authenticated? on-host? build-pipeline access?) frame every threat that cites the actor.

### Phase 2: Trust Boundaries & Data Flow Diagram

§ 3 Trust Boundaries inherits from INTERFACES § 1 — every boundary there appears here with added security annotations (authority change, validation point citing `EP-name` or component, data classification in transit). Internal boundaries between two components in different security zones (user-space vs privileged, tenant-scoped worker vs shared scheduler) are legitimate and must be listed; omission is the commonest modelling failure.

§ 4 is a Mermaid `flowchart LR`. Nodes are components (ARCHITECTURE § 2), data stores (DATA or ARCHITECTURE), and external entities (§ 2 Actors). Edges carry data-classification labels. Every trust-boundary crossing is annotated `⚡ trust-boundary: {name}`. A one-paragraph prose walkthrough names every `⚡` edge — un-called-out crossings drift into unnoticed boundaries.

### Phase 3: STRIDE-Driven Threat Enumeration

§ 5 applies STRIDE systematically to every (DFD element × category) cell. Elements are processes, data flows, data stores, and external entities; categories are Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation. Every cell is either a `THREAT-NN` entry or an explicit one-line dismissal tied to the element's properties (`Repudiation dismissed — this element neither writes authoritative logs nor takes business-critical actions`). Silence on a cell is oversight, not dismissal.

Each `THREAT-NN` fills every Output Format field: element, STRIDE category, concrete attack scenario (never "might be"), attacker capability assumed, impact tied to § 1 assets, likelihood (low/medium/high with evidence-grounded rationale), severity (low/medium/high/critical grounded in asset CIA impact), and `MIT-NN` list. Likelihood rationale cites capability assumptions and observed threat intelligence — `"medium: automated credential-stuffing tooling is commodity; EP-auth-login is internet-facing with no edge rate limit today"` is a rationale; `"medium: seems plausible"` is not.

### Phase 4: Mitigation Catalogue

§ 6 enumerates every control. Each `MIT-NN` fills the Output Format template: addressed threats, concrete description, implementation location (ARCHITECTURE § 2 component + `U-NN` from WORK_UNITS when known — when unknown, state the component and surface the missing unit in § 16), verification method (automated test / manual audit / code review / penetration test), dependencies.

Defence-in-depth is expected: one threat may reference multiple `MIT-NN`, and one `MIT-NN` may address multiple threats. A `MIT-NN` with no implementation location is an aspiration, not a control. Orphan threats (no mitigation) must be explicitly accepted in § 14 with compensating controls.

### Phase 5: Authentication & Authorization Models

§ 7 Authentication cites the scheme from INTERFACES § 3 verbatim and pins token lifetime, refresh rotation, revocation semantics, MFA strategy (required actors, factor, bypass/recovery), per-surface sign-in flow citing IA flow IDs (`WEB_IA § {flow-id}`), and session handling (storage, rotation on privilege change, invalidation triggers). Mobile and browser considerations carry distinct bullets — refresh-token rotation differs materially.

§ 8 Authorization enumerates the role inventory (explicit names), names the permission model (RBAC / ABAC / policy-based with justification tied to the domain), lists enforcement points (middleware, service layer, domain-service guards, DB row-level policies), and states delegation/impersonation rules. The use-case-to-permission map is the completeness rule: every `UC-NN` appears with required permissions or the explicit annotation `public — no authorization required`. Missing rows indicate either a missing use case (surface in § 16) or a hole in authorization (surface as critical finding).

### Phase 6: Secrets, Validation, Output Encoding

§ 9 covers per-secret-class storage (env var, secret manager, KMS — cite OPERATIONS or surface as open question), rotation policy (duration, trigger, automation), read/rotate/decrypt principal lists, and audit trail (cite `EVT-name` from QUALITY § 2 when available). Cover at minimum service-to-service credentials, encryption keys, webhook signing keys, third-party API tokens.

§ 10 states per-trust-boundary input validation (schema validation citing INTERFACES § 6 fields; security-specific field rules beyond schema; canonicalization of paths, URLs, identifiers, and encoded forms) and per-sink output encoding (HTML context-aware, JSON structured, SQL parameterised, shell forbidden or explicit execve-arg-array redirect).

### Phase 7: Abuse Prevention, Supply Chain, Incident Response

§ 11 covers rate limits (cite INTERFACES § 6 where pinned), content abuse, account abuse (sign-up rate limits, disposable-email detection, HIBP checks), CAPTCHA/anomaly detection, and automated-traffic detection. Each subsection is a concrete mechanism or explicit `Not applicable because {reason}` — silence is ambiguity.

§ 12 covers dependency update policy, lockfile commitment, SBOM generation, SAST/DAST/SCA roles (one bullet per category naming the role, not the product), and signature verification (code signing, provenance, SLSA level).

§ 13 has the six-stage lifecycle (detection → triage → containment → eradication → recovery → lessons) with signals, owners, and exit criteria per stage; paging rules citing QUALITY § 6; evidence preservation rules stating retention; and external disclosure (regulator notification windows, customer thresholds, public disclosure norms).

### Phase 8: Residual Risks, Relationship, Open Questions, Validation

§ 14 is the explicit accept-list. Each row: risk (citing `THREAT-NN`), why accepted (cost/timeline/external constraint — never "hard to fix"), compensating controls, review date within one year.

§ 15 is a fixed-order relationship list (ARCHITECTURE, DOMAIN, INTERFACES, IAs, USE_CASES, ERRORS, DATA, OPERATIONS, QUALITY, BEHAVIOR, SPEC-level).

§ 16 is the escape valve. Typical questions: "Do we require MFA for admin actors in v1?", "Is the internal service-to-service hop a trust boundary or do we assume mutual trust within the VPC?", "What residual-risk acceptance is appropriate for tenant-scoped queue starvation?".

Before finalising verify:

- Every `THREAT-NN` in § 5 has ≥ 1 `MIT-NN` reference in § 6 or is explicitly accepted in § 14 Residual Risks.
- Every `MIT-NN` in § 6 names an implementation location (component + work unit `U-NN` when known; otherwise the component alone with a § 16 entry flagging the missing unit).
- Every trust boundary in INTERFACES § 1 has a § 3 entry and a `⚡ trust-boundary: {name}` crossing in § 4's DFD.
- Every DFD element × STRIDE-category cell is either a `THREAT-NN` entry or carries an explicit dismissal with one-line justification.
- Every `UC-NN` from USE_CASES § 2 appears in the § 8 use-case-to-permission map (or is explicitly marked `public — no authorization required`).
- Every authorized actor in § 2 matches an actor in USE_CASES § 1 exactly.
- Every `EP-name`, `EVT-name`, `ERR_CODE`, `UC-NN`, `ADR-NN`, and `INV-NN` citation resolves to the named registry.
- § 14 Residual Risks has compensating controls and review dates for every row.
- Frontmatter counts match the body exactly. `status` is `complete` if § 16 reads "All questions resolved.", `has_open_questions` otherwise.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Stable IDs forever

`THREAT-NN` and `MIT-NN` IDs are assigned once and never renumbered, never reused, never silently deleted. Retired entries retain their row with `(retired — superseded by {new-id})` markers so downstream citations (`SPEC.md` files that cite `THREAT-42` and `MIT-17`) keep resolving. Numeric gaps in the sequence are expected and correct.

### 2. Every threat has a mitigation or an accepted residual

Every `THREAT-NN` in § 5 has at least one `MIT-NN` in its Mitigations field, OR an explicit reference to a § 14 Residual Risks row that accepts the threat. An orphan threat is a modelling failure — it reads as a claim that the system is vulnerable and no one has taken responsibility.

### 3. Every mitigation names an implementation location

Every `MIT-NN` in § 6 names a concrete implementation location: a component from ARCHITECTURE § 2 and — when the work unit is known — a `U-NN` from WORK_UNITS.md. A control with no owner is not a control. If the work unit is not yet decomposed, state the component and add a § 16 entry: `Which unit owns MIT-NN once /WORK_UNITS.md runs?`.

### 4. Every trust boundary from INTERFACES has a § 3 entry and a DFD crossing

INTERFACES.md § 1 is authoritative for boundaries. Every row there is a row in § 3 here, and every such boundary appears as a `⚡ trust-boundary: {name}` annotation on at least one edge in § 4's DFD. Missing a boundary is the silent-elision failure that STRIDE analysis exists to detect.

### 5. Every STRIDE cell is examined

For every DFD element in § 4, every STRIDE category (Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation) is either a `THREAT-NN` entry or an explicit one-line dismissal in § 5 under the element. Silence on a cell is not dismissal — it is oversight. Dismissal reads `Spoofing dismissed — this element has no caller identity to impersonate; it is the trust root.`, not `Spoofing: N/A`.

### 6. Every authorized actor traces to USE_CASES § 1 exactly

§ 2 Authorized Actors is a one-to-one map of USE_CASES § 1 Actors. Never invent an actor here that the use-case model does not know; never drop an actor the use-case model contains. If an authorization-relevant class exists (system process, admin tool, automated integration) that the use-case model lacks, surface the gap in § 16 and raise it back to `/use-cases` — do not patch it silently in § 2.

### 7. Every use case has a permissions row in § 8

Every `UC-NN` from USE_CASES § 2 appears in the § 8 use-case-to-permission map. The row is either `{UC-NN} — requires {permission list}` or `{UC-NN} — public (no authorization required)`. Missing rows indicate either a missing use case (re-enter `/use-cases`) or an unanalyzed hole in authorization (raise as critical `THREAT-NN`).

### 8. Evidence-grounded threats only

Every `THREAT-NN` carries a concrete attack scenario and a capability assumption. "Might be exploited" is not evidence; "An anonymous attacker can submit 1000 login attempts per second against `EP-auth-login` because no rate limit is enforced at the edge" is. If a scenario cannot be written, the threat has not yet been modelled — refine or drop.

### 9. Authentication and authorization cite INTERFACES and USE_CASES

§ 7 Authentication cites the scheme from INTERFACES § 3 verbatim — never redefines it. § 8 Authorization cites `UC-NN` from USE_CASES for every permission row. Redefinition creates contradiction bugs when INTERFACES or USE_CASES change; citation resolves them.

### 10. No implementation syntax

No code snippets (try / catch / async / await), no library or framework product names beyond what is already pinned elsewhere (`bcrypt` is allowed because it is a primitive; `crypto-js version X.Y` is not — that is `/spec`). Specific RBAC library choices, ORM-level row-level-policy product names, and secret-manager product names are OPERATIONS' concerns. SECURITY names the control; downstream owns the implementation.

### 11. No wire-schema design

SECURITY names threats, mitigations, auth schemes, and permission maps. It does not design HTTP request/response field names (INTERFACES), error code strings (ERRORS), or database column names (DATA). A SECURITY.md that enumerates JSON field shapes for security endpoints is out of scope — move the schema to INTERFACES and cite the `EP-name` here.

### 12. Residual risks have compensating controls and review dates

Every row in § 14 carries a compensating control (a lesser control that reduces likelihood or impact) and a review date no further than one year from the document date. Perpetual acceptance without review is an abandoned risk, not an accepted one.

### 13. Single YAML frontmatter block

One YAML block at the top containing common fields (`skill`, `date`, `status`) and security-specific counts (`assets`, `trust_boundaries`, `threats`, `mitigations`, `residual_risks`, `open_questions`). Never emit a second block. Counts match the body exactly.

### 14. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.", "various", "and so on", "industry-standard", "best practice". Use exact IDs, exact mechanism names, exact durations (`15 minutes`, not `short window`), exact numeric thresholds (`100 requests per minute per IP`, not `reasonable rate limit`). Unresolvable ambiguity surfaces in § 16 Open Questions with options, tradeoffs, and a recommendation.

---

## Output Format

```markdown
---
skill: SECURITY.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
assets: {N}
trust_boundaries: {N}
threats: {N}
mitigations: {N}
residual_risks: {N}
open_questions: {N}
---

# SECURITY — {ProductName}

> Consolidated threat model and controls. Every asset, hostile actor, trust
> boundary, STRIDE threat, mitigation, authentication and authorization rule,
> secret policy, validation requirement, abuse-prevention control, supply-chain
> policy, incident-response step, and accepted residual risk lives here.
> Downstream artifacts cite by stable `THREAT-NN` and `MIT-NN` IDs. Error code
> definitions live in ERRORS.md; wire shapes in INTERFACES.md; runbook contents
> and secret-manager product choices in OPERATIONS.md; domain entities in DOMAIN.md.

## § 1. Assets

### `{AssetName}`

- **CIA classification:** confidentiality `{low / medium / high}`, integrity `{low / medium / high}`, availability `{low / medium / high}`
- **Location:** `{table / file path / memory in ComponentName / external system}` (cite ARCHITECTURE § {N})
- **Owning context:** `{ContextName from DOMAIN § 2}`
- **Exposure:** `{list of components from ARCHITECTURE § 2 that can read / write this asset}`
- **Related domain entity:** `{EntityName from DOMAIN § 4}` or `(not a domain entity — this is a system-level asset)`

(Repeat for every asset. Alphabetical within each confidentiality tier. If none beyond trivial public assets: document the single asset and note the scope.)

---

## § 2. Actors

### § 2.1. Authorized actors

| Actor | USE_CASES reference | Authentication level | Primary use cases |
|-------|---------------------|---------------------|-------------------|
| `{ActorName}` | `USE_CASES § 1` | `{anonymous / authenticated / admin}` | `UC-{NN}`, `UC-{NN}` |

(One row per actor from USE_CASES § 1. No additions, no omissions.)

### § 2.2. Unauthorized / hostile classes

### `{HostileClassName — e.g., "Anonymous external attacker"}`

- **Capability assumption:** `{network-only / authenticated as low-privilege user / on-host / build-pipeline access / nation-state observer}`
- **Motivation:** `{financial / disruption / data exfiltration / ransom / espionage}`
- **Assumed resources:** `{single-machine / botnet / dedicated infrastructure / state-actor}`
- **Typical attack patterns:** `{short list of patterns this class uses}`

(Repeat for every hostile class relevant to the system. At minimum: anonymous
 attacker, authenticated-but-unauthorized user, compromised dependency. Add
 malicious insider, supply-chain attacker, nation-state when they apply.)

---

## § 3. Trust Boundaries

### `{BoundaryName — matches INTERFACES § 1}`

- **Between:** `{Component A / actor} ↔ {Component B}` (cite ARCHITECTURE § {N})
- **Authority change:** `{anonymous user → authenticated user / tenant A → tenant B / user-space → privileged service}`
- **Validation point:** `{where crossing traffic is authenticated, authorized, and schema-validated — EP-name or component + middleware}`
- **Data classification in transit:** `{PII / secret / internal / public}` — aligns with § 1 asset classifications
- **Related threats:** `THREAT-{NN}`, `THREAT-{NN}`

(Repeat for every boundary from INTERFACES § 1. Add internal boundaries that
 INTERFACES § 1 does not enumerate when two components run in distinct security
 zones — and raise those additions as open questions to `/interfaces`.)

---

## § 4. Data Flow Diagram

```mermaid
flowchart LR
    Actor[{ActorName}] -- "{data classification}" --> EdgeComponent[{Component A}]
    EdgeComponent -- "⚡ trust-boundary: {BoundaryName} / {data classification}" --> InternalService[{Component B}]
    InternalService -- "{data classification}" --> DataStore[({DataStoreName})]
```

{One-paragraph prose walking through every `⚡` edge: which boundary it crosses,
 what validation happens, and which threats target the crossing.}

---

## § 5. Threats

Organized by DFD element. Within each element, a subsection per STRIDE category
contains either `THREAT-NN` entries or an explicit dismissal.

### Element: `{ComponentName / DataFlow / DataStoreName / ExternalEntity}`

#### Spoofing

#### `THREAT-{NN}: {short title}`

- **Element:** `{ComponentName / DataFlow A→B / DataStoreName / ExternalEntity}`
- **STRIDE category:** `Spoofing`
- **Description:** `{concrete attack scenario — e.g., "An anonymous attacker replays a captured bearer token against EP-push-create within its 15-minute TTL, posing as the captured token's owner."}`
- **Attacker capability assumed:** `{from § 2.2 — e.g., "Anonymous attacker with network position to capture TLS-terminated traffic at a compromised intermediary"}`
- **Impact:** `{asset impact — e.g., "Integrity (high) and Confidentiality (medium) of the victim's Repository asset: attacker can push malicious commits and read private contents."}`
- **Likelihood:** `{low / medium / high}` — `{rationale grounded in capability assumptions and observed threat intelligence}`
- **Severity:** `{low / medium / high / critical}` — `{rationale grounded in asset CIA impact}`
- **Mitigations:** `MIT-{NN}`, `MIT-{NN}`

(Repeat per threat in this STRIDE category. If none: `Spoofing dismissed — {one-line justification}`.)

#### Tampering

(As above.)

#### Repudiation

(As above.)

#### Information Disclosure

(As above.)

#### Denial of Service

(As above.)

#### Elevation of Privilege

(As above.)

(Repeat for every DFD element. Every STRIDE category under every element must
 be either enumerated threats or an explicit dismissal.)

---

## § 6. Mitigations

### `MIT-{NN}: {short title}`

- **Addresses threats:** `THREAT-{NN}`, `THREAT-{NN}`
- **Description:** `{concrete control — e.g., "Schema-validate every request body against the INTERFACES § 6 shape for EP-name at the API gateway before the handler dispatches; reject with ERR_BAD_REQUEST on any field-level failure."}`
- **Implementation location:** `{ComponentName from ARCHITECTURE § 2}` (owned by `U-{NN}` from WORK_UNITS.md, or `(unit unknown — surfaced in § 16)`)
- **Verification:** `{automated test / manual audit / code review / penetration test — name the specific check}`
- **Dependencies:** `{libraries, services, config vars — name primitives only, not product choices}`

(Repeat per mitigation. Alphabetical by `MIT-NN` number. Retired mitigations
 retain their row with `(retired — superseded by MIT-{NN})`.)

---

## § 7. Authentication Model

- **Scheme:** `{Bearer JWT / session cookie / mTLS / HMAC-signed request}` — cite `INTERFACES § 3`
- **Token lifetime:** `{duration — e.g., 15 minutes access token, 24 hours refresh token}`
- **Refresh semantics:** `{rotation policy — e.g., refresh token rotates on every refresh; compromise detection via token family revocation}`
- **Revocation:** `{how tokens are revoked — blocklist with TTL, short-lived access tokens only, explicit revocation endpoint EP-name}`
- **MFA strategy:**
  - Required for: `{actor list — e.g., "admin, account owner"}`
  - Factor: `{TOTP / WebAuthn / SMS with risk-based escalation only}`
  - Bypass: `{recovery flow — e.g., "support-assisted with identity verification; audit-logged as EVT-mfa-bypass"}`
- **Per-surface sign-in flow:**
  - `{SurfaceName}`: cite `{IA § flow-id}` (e.g., `WEB_IA § F-signin`)
- **Session handling:**
  - Storage: `{where session state lives — httpOnly secure cookie / KV store with session ID / pure JWT stateless}`
  - Rotation: `{on password change, on privilege elevation, on idle timeout of N minutes}`
  - Invalidation: `{explicit sign-out / account lock / password reset / suspicious-activity detection}`

---

## § 8. Authorization Model

- **Roles:** `{comma-separated role names — "owner", "member", "viewer", "admin"}`
- **Permission model:** `{RBAC / ABAC / policy-based}` — `{one-paragraph justification — why this model fits the domain}`
- **Enforcement points:**
  - `{Component + layer — e.g., "API middleware: coarse role check for sev1 operations"}`
  - `{Component + layer — e.g., "Domain-service guards: fine-grained ABAC checks citing entity ownership"}`
  - `{Component + layer — e.g., "DB row-level policies: defence-in-depth last line"}`
- **Delegation / impersonation:**
  - `{Rules — e.g., "Admins may impersonate members for support; every impersonated session carries a Delegate-By header and emits EVT-impersonation-started; impersonation sessions cannot perform destructive admin actions."}`
  - `(Or: "No delegation or impersonation supported in v1.")`
- **Use-case-to-permission map:**

| Use case | Required permissions | Enforcement point |
|----------|---------------------|-------------------|
| `UC-{NN}` | `{permission list / "public — no authorization required"}` | `{enforcement point from the list above}` |

(Every `UC-NN` from USE_CASES § 2 appears. No omissions.)

---

## § 9. Secret & Credential Management

### `{SecretClass — e.g., "Service-to-service JWT signing key"}`

- **Storage:** `{env var / secret manager — cite OPERATIONS § {N} when produced}`
- **Rotation policy:** `{duration / trigger / automation status — e.g., "90-day rotation; automated via secret-manager built-in rotation; incident-triggered emergency rotation tested quarterly"}`
- **Principals with read access:** `{list of roles / services}`
- **Principals with rotate access:** `{list of roles / services}`
- **Audit trail:** `{how access is logged — EVT-name from QUALITY § 2 or "no audit trail yet — § 16"}`

(Repeat per secret class. Cover at minimum: service-to-service credentials,
 encryption keys, webhook signing keys, third-party API tokens.)

---

## § 10. Input Validation & Output Encoding

### Input validation per trust boundary

### `{BoundaryName from § 3}`

- **Schema validation:** `{what is validated at entry — cite INTERFACES § 6 field catalogue}`
- **Canonicalization:** `{rules for paths, URLs, identifiers, encoded forms, Unicode}`
- **Security-specific field rules beyond schema:** `{e.g., "Email canonicalises to RFC 5321; trailing whitespace rejected; plus-addressing permitted; consecutive dots rejected."}`

(Repeat per boundary from § 3.)

### Output encoding per sink

- **HTML:** `{context-aware encoding rules — element text, attribute, JS string, URL, CSS}`
- **JSON:** `{rules — e.g., "No string interpolation; structured serialisation only."}`
- **SQL:** `{rules — e.g., "Parameterised queries mandatory; ORM verbatim when used; string concatenation forbidden in query construction."}`
- **Shell:** `{rules — e.g., "Shell interpolation forbidden in production code paths; if required, use execve with arg array, never shell string."}`

---

## § 11. Abuse Prevention

- **Rate limits:** per endpoint cited in INTERFACES § 6 — `{summary: which endpoints carry limits, which are explicitly limit-free and why}`
- **Content abuse:** `{user-generated content filtering rules / media moderation / link scanning}`
- **Account abuse:** `{sign-up rate limits, disposable-email detection, password-reuse checks against HIBP}`
- **CAPTCHA / anomaly detection:** `{which surfaces, which triggers, which escalation — or "not applicable because {reason}"}`
- **Automated-traffic detection:** `{per-IP reputation, per-ASN throttling, JA3 fingerprinting — or "not applicable because {reason}"}`

---

## § 12. Supply-Chain Security

- **Dependency update policy:** `{auto-merge class / manual-review class / pinned critical dependencies}`
- **Lockfile commitment:** `{rule — e.g., "Cargo.lock / package-lock.json committed; CI enforces freshness on every PR."}`
- **SBOM generation:** `{tool family / frequency / retention}`
- **Vulnerability scanning:**
  - SAST: `{role — static analysis on every PR}`
  - DAST: `{role — dynamic scan on staging weekly}`
  - SCA: `{role — dependency vulnerability scan on every build}`
- **Signature verification:** `{release signing / provenance / SLSA level targeted}`

---

## § 13. Incident Response

- **Detection:** `{signals and ownership — cite QUALITY § 6 alerts when produced}`
- **Triage:** `{decision criteria for sev classification, owner rotation, stakeholder loop}`
- **Containment:** `{typical actions — isolate component, rotate credentials, disable endpoint, blocklist}`
- **Eradication:** `{root-cause fix path, verification that the attacker foothold is removed}`
- **Recovery:** `{restore-from-known-good procedures, gradual rollout, monitoring heightened}`
- **Lessons:** `{post-mortem cadence, tracking of action items, re-test policy}`
- **Paging rules:** `{severity → paging destination — cite QUALITY § 6 or surface as open question for /operations}`
- **Evidence preservation:** `{log / trace / artifact retention from first detection through post-mortem — state required duration}`
- **External disclosure:**
  - Regulator notification: `{e.g., "GDPR: 72 hours from awareness of a personal-data breach"}`
  - Customer notification: `{threshold and channel}`
  - Public disclosure: `{coordinated timing norms — e.g., 90-day responsible-disclosure window for dependency-sourced vulnerabilities}`

---

## § 14. Residual Risks

| Risk | Why accepted | Compensating controls | Review date |
|------|--------------|-----------------------|-------------|
| `{one-sentence risk, citing THREAT-NN}` | `{cost / timeline / external constraint — never "hard to fix"}` | `{the lesser controls that reduce likelihood or impact}` | `{YYYY-MM-DD, no further than one year from document date}` |

(Repeat per accepted residual risk. If none: "No residual risks accepted — every
 `THREAT-NN` is mitigated in § 6.")

---

## § 15. Relationship to Other Artifacts

- **ARCHITECTURE.md** owns components and `ADR-NN` decisions; every component in §§ 1, 3, 4, 6 exists in ARCHITECTURE § 2.
- **DOMAIN.md** owns entities and invariants; § 1 cites entity names verbatim; `INV-NN` is cross-referenced where invariant violation is itself a security event.
- **INTERFACES.md** owns boundaries and wire conventions; § 3 classifies every boundary from INTERFACES § 1; § 7 cites the auth scheme from INTERFACES § 3 verbatim.
- **IAs** define auth UX and error visibility; § 7 cites per-surface sign-in flow IDs.
- **USE_CASES.md** owns actors and use cases; § 2.1 is a one-to-one map of USE_CASES § 1; § 8 covers every `UC-NN`.
- **ERRORS.md** owns error code strings; §§ 5, 8, 11 reference security-signal codes by string.
- **DATA.md** owns the sensitive-column inventory; § 1 cites DATA § 8 rows; § 10 cites DATA validation rules.
- **OPERATIONS.md** owns runbook contents, secret-manager product choices, and alert routing; §§ 9, 13 reference it.
- **QUALITY.md** owns reliability and performance alerts; security-specific detection alerts are cataloged here, cross-referenced from QUALITY for routing.
- **BEHAVIOR.md** owns state machines and sagas; security-critical transitions are cross-referenced via `SM-{entity}: {from} → {to}` where relevant.
- **SPECs** for security-sensitive units cite `THREAT-NN` and `MIT-NN`; `/review` verifies the declarations; `/system-verify` exercises adversarial scenarios.

---

## § 16. Open Questions

- [ ] `{Question — e.g., "Do we require MFA for admin actors in v1, or defer to v2?"}`
  - **Option A:** `{description — e.g., "Require TOTP for admin in v1; ships with onboarding friction but hard-blocks account-takeover for privileged accounts."}` — `{tradeoff}`
  - **Option B:** `{description — e.g., "Defer MFA to v2; ship v1 with strong password policy and anomaly detection only."}` — `{tradeoff}`
  - **Recommendation:** `{suggestion and reasoning, grounded in asset CIA ratings from § 1}`

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Assets with CIA classifications, locations, owning contexts, and exposures
- Authorized actors (one-to-one with USE_CASES § 1) and hostile actor classes with capability assumptions
- Trust boundaries inherited from INTERFACES § 1, with authority changes, validation points, and data classifications
- Data-flow diagram with trust-boundary crossings annotated
- STRIDE-driven threat catalogue with stable `THREAT-NN` IDs, per-element × per-category coverage, and concrete attack scenarios
- Mitigation catalogue with stable `MIT-NN` IDs, concrete implementation locations, verification methods, and dependencies
- Authentication model (scheme, token lifetime, refresh, revocation, MFA, per-surface sign-in, session handling)
- Authorization model (roles, permission model, enforcement points, delegation, use-case-to-permission map)
- Secret and credential management (storage, rotation, principals, audit trail)
- Input validation per trust boundary and output encoding per sink
- Abuse prevention (rate limits, content abuse, account abuse, CAPTCHA, automated-traffic detection)
- Supply-chain security (dependency policy, lockfile, SBOM, SAST/DAST/SCA, signature verification)
- Incident response lifecycle with detection → triage → containment → eradication → recovery → lessons
- Residual risks register with compensating controls and review dates
- Cross-artifact relationships with ARCHITECTURE, DOMAIN, INTERFACES, IAs, USE_CASES, ERRORS, DATA, OPERATIONS, QUALITY, BEHAVIOR, SPEC-level
- Genuinely ambiguous security decisions surfaced in § 16 Open Questions

### Out of scope

- Implementation of security controls (validation code, bcrypt calls, JWT verification, rate-limit middleware) — owned by `/spec` and `/implement`.
- Log and alert implementation (log-shipper choice, alert-routing product syntax) — owned by `/quality` and `/operations`.
- Encryption-at-rest configuration (KMS key IDs, SSE-KMS policy documents) — owned by `/data` and `/operations`; this document states the requirement.
- Sign-in screen layouts and MFA prompt flows at pixel level — owned by the IA skills; this document names the flow by IA flow ID.
- External compliance audits (SOC2, ISO27001, HIPAA) — this document informs them; the audit workflow is not owned here.
- Wire shape of security endpoints (login, password reset, token refresh field names) — owned by `/interfaces`.
- Error code string definitions (`AUTHN_FAILED` → HTTP status → user message) — owned by `/errors`; this document cites codes by string.
- Database schema for security-relevant tables (sessions, audit logs) — owned by `/data`.
- Runbook contents for security incidents — owned by `/operations`; this document names the lifecycle only.
- Cost modelling of security controls — not part of the ADD workflow.

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `assets`, `trust_boundaries`, `threats`, `mitigations`, `residual_risks`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.", "various", "and so on", "industry-standard", "best practice", "reasonable")
- [ ] § 16 Open Questions is present (empty with "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files beyond the referenced registries (INTERFACES § 1, USE_CASES § 1–2, DOMAIN § 4, ARCHITECTURE § 2, ERRORS § 3)
- [ ] All sixteen sections § 1 through § 16 are present with their exact headings
- [ ] § 1 Assets lists every asset with CIA classification, location, owning context, exposure, and related domain entity (or explicit "not a domain entity" annotation)
- [ ] § 2.1 Authorized actors is a one-to-one map of USE_CASES § 1 — no additions, no omissions
- [ ] § 2.2 Hostile classes cover at minimum anonymous attacker, authenticated-but-unauthorized user, and compromised dependency; each carries an explicit capability assumption
- [ ] § 3 Trust Boundaries contains every boundary from INTERFACES § 1; any internal boundary added beyond INTERFACES is also raised as an open question to `/interfaces`
- [ ] Every § 3 entry names authority change, validation point, data classification, and related `THREAT-NN` IDs
- [ ] § 4 Data Flow Diagram is present as a Mermaid `flowchart LR` with every trust-boundary crossing annotated `⚡ trust-boundary: {BoundaryName}`, followed by a one-paragraph prose walkthrough
- [ ] § 5 Threats covers every (DFD element × STRIDE category) cell — either as `THREAT-NN` entries or as explicit one-line dismissals with justification
- [ ] Every `THREAT-NN` entry has element, STRIDE category, concrete attack scenario (no "might be"), attacker capability assumed, impact tied to assets, likelihood with rationale, severity with rationale, and a non-empty `Mitigations:` list (or an explicit reference to § 14 Residual Risks)
- [ ] Every `MIT-NN` in § 6 has addressed threats, description, implementation location (component + `U-NN` or explicit missing-unit annotation), verification method, and dependencies
- [ ] Every `THREAT-NN` has ≥ 1 `MIT-NN` reference OR an explicit § 14 Residual Risks row
- [ ] § 7 names scheme (cited from INTERFACES § 3), token lifetime, refresh, revocation, MFA strategy, per-surface sign-in flows (cited by IA flow ID), and session handling
- [ ] § 8 names the role inventory, permission model with justification, enforcement points, delegation rules, and a use-case-to-permission row for every `UC-NN` from USE_CASES § 2
- [ ] § 9 has a block per secret class covering storage, rotation policy, read principals, rotate principals, and audit trail — at minimum service-to-service credentials, encryption keys, webhook signing keys, third-party API tokens
- [ ] § 10 has input validation per trust boundary and output encoding rules for HTML, JSON, SQL, and shell sinks
- [ ] § 11 covers rate limits (citing INTERFACES § 6), content abuse, account abuse, CAPTCHA / anomaly detection, and automated-traffic detection — each bullet either concrete mechanism or explicit "not applicable because {reason}"
- [ ] § 12 covers dependency update policy, lockfile commitment, SBOM generation, SAST/DAST/SCA roles, and signature verification
- [ ] § 13 covers all six stages (detection, triage, containment, eradication, recovery, lessons) plus paging rules, evidence preservation, and external disclosure
- [ ] § 14 Residual Risks has a compensating control and a review date (within one year of document date) for every row
- [ ] § 15 Relationship to Other Artifacts names ARCHITECTURE, DOMAIN, INTERFACES, IAs, USE_CASES, ERRORS, DATA, OPERATIONS, QUALITY, BEHAVIOR, and SPEC-level artifacts
- [ ] Every `THREAT-NN` and `MIT-NN` ID is unique and stable; retired IDs retain their row with `(retired — superseded by {new-id})`
- [ ] Every `EP-name` cited exists in INTERFACES § 6; every `EVT-name` cited exists in INTERFACES § 7 or QUALITY § 2; every `ERR_CODE` cited exists in ERRORS § 3; every `UC-NN` cited exists in USE_CASES § 2; every component cited exists in ARCHITECTURE § 2; every entity cited exists in DOMAIN § 4; every `INV-NN` cited exists in DOMAIN § 9
- [ ] No implementation syntax (grep-check: `try`, `catch`, `async`, `await`, `goroutine`, `thread`, `mutex`) and no wire-schema design (grep-check: `JSON body`, `camelCase`, `snake_case` as field-name design, `column`, `table`)
- [ ] No product-name choices beyond primitives (grep-check: `Vault`, `AWS Secrets Manager`, `GCP Secret Manager`, `Cloudflare`, `Okta`, `Auth0`, `PagerDuty`, `Snyk`, `Dependabot`) — primitives like `bcrypt`, `TLS`, `JWT` are allowed because they are standards, not products
- [ ] Frontmatter counts match the body: `assets`, `trust_boundaries`, `threats`, `mitigations`, `residual_risks`, `open_questions`
- [ ] `status` is `complete` if § 16 is "All questions resolved." and `has_open_questions` otherwise
- [ ] Document length is within the 500–1500 line target (hard cap 2000)
