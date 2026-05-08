# Design-Review Checklist — Per-Artifact, Pairwise, and Coverage Probes

This reference file is read by the `/design-review` skill during Phase 2 (per-artifact quality), Phase 3 (pairwise consistency), Phase 4 (completeness), and Phase 5 (cross-reference matrix construction). It organises the systematic probes that produce `QF-NN`, `CF-NN`, `MF-NN` findings and feeds § 5 of DESIGN_REVIEW.md.

The SKILL.md body describes **how** to apply these probes in phases and how to aggregate findings. This file owns **what** each probe looks for.

Structure:

1. Cross-reference index recipe (feeds Phase 1 extraction and Phase 5 matrix)
2. Per-artifact quality probes (feeds Phase 2 → `QF-NN`)
3. Pairwise consistency probes (feeds Phase 3 → `CF-NN`)
4. Completeness probes (feeds Phase 4 → `MF-NN`)
5. Severity calibration heuristics (feeds every phase)

Walk every section end-to-end on every review. Silent omission of a probe is the failure mode this file exists to prevent.

---

## 1. Cross-reference index recipe

The Phase 1 extraction scans the eight required artifacts and produces an in-agent index of every definition and every citation. Phase 5 then emits § 5 of the output.

### Prefixes to extract

| Prefix pattern | Authoritative artifact | Section (typical) | Notes |
|----------------|------------------------|-------------------|-------|
| `INV-NN` | DOMAIN | § 9 Invariants Index | Numeric gaps allowed if retired; any retired row must say so |
| `ENT-NN` | DOMAIN | § 4 Entities / Aggregates | Rare — entities usually cited by name |
| `EP-{kebab}` | INTERFACES | § 6 Endpoints | Names are kebab-case verbs/nouns (`push-create`, `user-login`) |
| `EVT-{kebab}` | INTERFACES | § 7 Events | Some projects define domain events in DOMAIN § 7 — matrix must resolve to exactly one definer |
| `ERR_{SHOUTY}` | ERRORS | § 3 Error Code Registry | Underscore style, SHOUTY_SNAKE_CASE |
| `SM-{entity}-{state}` or `SM-{entity}: {from} → {to}` | BEHAVIOR | § 1 State Machines | Two forms coexist; matrix treats them as the same prefix |
| `SAGA-{kebab}` | BEHAVIOR | § 2 Sagas | Single flat namespace |
| `METRIC-{kebab}` | QUALITY | § 3 Metrics Catalogue | Units in the name are conventional (`-ms`, `-count`, `-ratio`) |
| `SLO-{kebab}` | QUALITY | § 4 SLOs | Every SLO references METRIC-* for its SLI |
| `ALERT-{kebab}` | QUALITY | § 6 Alerts | Cited from OPERATIONS § 7 runbooks; OPERATIONS is out of scope here |
| `THREAT-NN` | SECURITY | § 5 Threats | Sequential; gaps retained with `(retired — …)` |
| `MIT-NN` | SECURITY | § 6 Mitigations | Sequential; must name implementation location |
| `UC-NN` | USE_CASES | § 2 Use Cases | Only indexed if USE_CASES.md was read |
| `ADR-NN` | ARCHITECTURE | inline decision records | May appear in multiple sections of ARCHITECTURE |
| `CFG_{SHOUTY}` | OPERATIONS | § 4 Config Catalogue | **Out of scope** for this review — mark citing rows as `OUT-OF-SCOPE` in § 5, not `MISSING` |
| `RUNBOOK-{kebab}` | OPERATIONS | § 7 Runbook | Same treatment as CFG_ |
| `SCN-{kebab}` | surface IAs | cross-surface scenarios | Only indexed if an IA was read |

### Matrix-building rules

- One row per **unique citation site** (not per unique ID) — if `EP-push` is cited from BEHAVIOR § 2 and from SECURITY § 8, it produces two rows.
- Status values: `OK`, `MISSING`, `AMBIGUOUS` (multiple incompatible definitions), `OUT-OF-SCOPE` (references an artifact not in this review's read set — typically OPERATIONS or per-unit SPEC).
- Prefixes with zero citations across the suite get exactly one placeholder row: `"No {PREFIX}-* citations in the suite."` — this proves the scan happened.
- A citation that resolves to the wrong artifact (e.g., `EP-push` defined in QUALITY instead of INTERFACES) is `AMBIGUOUS`, not `OK`.

### Common extraction mistakes

- **Missing the retirement form.** A retired ID still defines the namespace position; treat `(retired — superseded by MIT-42)` as a valid definition for the purpose of the matrix.
- **Confusing event registries.** Wire events (INTERFACES § 7) and domain log events (QUALITY § 2 or DOMAIN § 7, project-specific) sometimes share the `EVT-*` prefix. If both exist, the matrix must distinguish them by citing the defining section; if the project's convention disallows this overlap, the overlap itself is a `CF-NN` blocking finding (naming collision).
- **Component names are not IDs.** `APIGateway`, `AuthService` are cited by name, not by a stable prefix. Component-name consistency is checked by the pairwise probes (DOMAIN ↔ ARCHITECTURE), not by the matrix.

---

## 2. Per-artifact quality probes

Apply each sub-section's probes to the named artifact. Each hit produces a `QF-NN` finding.

### DOMAIN.md

- Every entity name appears in § 1 Glossary and is used consistently in §§ 4–6 (spelling drift: `Repository` vs `Repo`).
- Every `INV-NN` is numbered sequentially (gaps allowed only with `(retired — …)` rows).
- Every bounded context has at least one entity assigned; orphan contexts (named but empty) are `QF` findings.
- Every value object carries an invariant (format, length, encoding) — value objects with no rules are thin wrappers and produce a `QF` medium.
- Frontmatter `entities`, `invariants`, `contexts` counts match the body.
- Placeholder language scan: `appropriate`, `relevant`, `reasonable`, `etc.`, `various`.
- Unresolved open questions in § 10 (or equivalent) when `status: complete` in frontmatter.

### ARCHITECTURE.md

- Every `ADR-NN` has Context, Decision, Alternatives Considered, Consequences — missing fields are `QF` high.
- Every component in § 2 has a responsibility, an owner, and runs on a runtime — thin entries (name-only) are `QF` medium.
- Every cross-component flow names the trigger, the sequence, and the failure mode — flows with no failure-mode row are `QF` high (downstream `/behavior` has nothing to base the saga on).
- The deployment-shape section (typical § 6) names the minimum viable deployment — absent deployment shape is a `QF` high blocking `/operations` downstream.
- Frontmatter `components`, `flows`, `adrs` counts match the body.

### INTERFACES.md

- § 1 Boundaries lists every boundary named by SECURITY § 3 (cross-check) and every integration named by OPERATIONS § 6 when OPERATIONS exists (cross-check later).
- Every `EP-name` has request shape, response shape, `possible_errors` list, idempotency behaviour, rate limits, and auth requirement — missing fields are `QF` high (per-field severity: missing `possible_errors` is blocking, missing rate limit is high).
- Every `EVT-name` has payload schema, producer, consumer list, delivery semantics — missing consumer list is `QF` high.
- § 2 naming convention is declared once and followed across every field in § 6 / § 7 — field-naming drift is a `CF-NN` not a `QF-NN` (cross-section, therefore cross-artifact-style inconsistency inside one artifact).
- § 3 Authentication is cited verbatim by SECURITY § 7 — verify the citation in Phase 3 pairwise probe.
- Frontmatter `endpoints`, `events`, `boundaries` counts match the body.

### DATA.md

- Every table has columns with explicit types, nullability, defaults, and constraints — missing constraints are `QF` high (DOMAIN invariant enforcement probe picks them up in Phase 3).
- Every table has a named owner (bounded context from DOMAIN § 2) — orphan tables are `QF` high.
- Every sensitive column in § 8 (or equivalent) has a classification (PII / secret / internal / public) — unclassified sensitive columns are `QF` blocking (SECURITY and QUALITY log-redaction depend on it).
- Access patterns per table name the read paths, write paths, and hot indexes — absent access patterns are `QF` medium.
- Migration policy names the tool family and the forward-only/reversible rule — absent is `QF` high blocking `/operations`.

### BEHAVIOR.md

- Every state machine `SM-{entity}: *` covers every DOMAIN entity that has a lifecycle — missing state machines for aggregates-with-lifecycle are `QF` blocking (this also surfaces as a completeness finding against DOMAIN).
- Every transition carries guard conditions and side-effect annotations — absent guards are `QF` high.
- Every `SAGA-name` has steps, compensations, idempotency key, timeout, and an originating event or endpoint — absent idempotency key is `QF` blocking.
- § 6 Event Emission Timing covers every `EVT-name` defined in INTERFACES § 7 — silent gaps are pairwise findings (Phase 3), but missing subsection structure is a `QF` high.
- Frontmatter `state_machines`, `sagas`, `transitions` counts match the body.

### QUALITY.md

- Every `METRIC-*` has type (counter / gauge / histogram / summary), unit, labels with cardinality, owner — thin metrics with no cardinality declared are `QF` high.
- Every `SLO-*` has SLI formula citing specific `METRIC-*` IDs, window, target, burn-rate alerts — SLOs with no burn-rate alert are `QF` blocking (`/operations` runbook has no routing).
- § 7 Performance Budgets has per-flow p50/p95/p99 that sum with an explicit overhead — budgets with unaccounted slack are `QF` medium.
- § 10 Back-Pressure names the load-shedding ladder — absent is `QF` high.
- Frontmatter counts match.

### SECURITY.md

- Every `THREAT-NN` has concrete attack scenario, capability assumption, impact, likelihood-with-rationale, severity-with-rationale, and non-empty Mitigations list OR an explicit residual-risk row — orphan threats are `QF` blocking.
- Every `MIT-NN` names an implementation location (component + `U-NN` or explicit `(unit unknown — § 16)`) — orphan mitigations are `QF` blocking.
- Every STRIDE cell (DFD element × category) is either a threat or an explicit dismissal — silent cells are `QF` high.
- § 7 authentication scheme cites INTERFACES § 3 verbatim (Phase 3 pairwise probe verifies this).
- § 8 use-case-to-permission map covers every `UC-NN` when USE_CASES is read — missing rows are pairwise / completeness findings (Phase 3 / Phase 4).
- Every § 14 Residual Risks row has compensating controls and a review date within one year — absent either field is `QF` high.

### ERRORS.md

- Every `ERR_CODE` has HTTP status, user-facing message, developer-facing description, recoverable / non-recoverable classification — missing recoverability classification is `QF` high.
- Every code appears in at least one endpoint's `possible_errors` in INTERFACES § 6 (pairwise probe), one saga's error path in BEHAVIOR § 2 (pairwise probe), or an explicit "internal only" annotation.
- Retired codes retain their row — silent deletion is `QF` blocking (existing SPECs may cite the retired code).
- Frontmatter `codes`, `http_statuses` counts match.

---

## 3. Pairwise consistency probes

Each subsection lists the ordered probes for one pair. Every hit produces a `CF-NN` finding.

### DOMAIN ↔ ARCHITECTURE

- Every bounded context in DOMAIN § 2 maps to at least one component in ARCHITECTURE § 2 — orphan contexts are `CF` high.
- Every component in ARCHITECTURE § 2 names an owning context that exists in DOMAIN § 2 — orphan components are `CF` high.
- Entity names used in ARCHITECTURE cross-component flows are spelled as DOMAIN § 1 defines them — spelling drift is `CF` medium (blocking if the misspelled name becomes a field name elsewhere).
- `ADR-NN` decisions do not contradict DOMAIN invariants — an ADR that accepts eventual consistency while DOMAIN `INV-7` demands a strong-consistency uniqueness is `CF` blocking.

### DOMAIN ↔ DATA

- Every DOMAIN aggregate root has a corresponding primary table or clearly-annotated view — orphan aggregates are `CF` blocking (`/spec` has no persistence target).
- Every `INV-NN` of the form "X is unique", "Y is non-negative", "Z is always present" is enforced by a DATA constraint — UNIQUE, CHECK, NOT NULL — on the relevant column. Missing enforcement is `CF` blocking.
- Every value object's format rule is enforced by a DATA check constraint or an explicit "enforced only at API boundary" annotation — silent reliance on application-layer validation is `CF` high.
- Sensitive columns declared in DATA § 8 match DOMAIN value objects that carry sensitivity (emails, phone numbers, tokens, secrets) — mismatches are `CF` high.

### DOMAIN ↔ BEHAVIOR

- Every DOMAIN aggregate with a lifecycle has a state machine in BEHAVIOR § 1 — missing state machine is `CF` high (also caught by Phase 4 completeness against `/spec`).
- Every state-machine transition preserves the relevant DOMAIN invariants — a transition that leaves the aggregate in a state violating `INV-N` is `CF` blocking.
- Domain events named in DOMAIN (if the project has a § 7 domain events subsection) match `EVT-*` in BEHAVIOR emission timing — drift is `CF` high.

### INTERFACES ↔ DOMAIN

- Every `EP-name` endpoint name and request/response field uses DOMAIN vocabulary — `/repositories/{repoId}/push` is valid; `/repos/{id}/push` when DOMAIN says `Repository` is `CF` medium.
- Every endpoint authorised action maps to a DOMAIN-understood operation — endpoints that mutate state outside DOMAIN's model are `CF` blocking (the domain is incomplete or the endpoint is invented).

### INTERFACES ↔ BEHAVIOR

- Every `EP-name` mutating endpoint is the entry point for exactly one saga OR an explicit "single-step, no saga" annotation — ambiguous ownership is `CF` high.
- Every `EVT-name` declared in INTERFACES § 7 is emitted by at least one BEHAVIOR saga step OR one state-machine transition — orphan events are `CF` blocking (consumers have no producer).
- Every `SAGA-name` has an originating `EP-name` or an explicit `"scheduler-driven"` annotation — orphan sagas are `CF` high.
- Idempotency keys declared in INTERFACES § 4 match the keys used by BEHAVIOR sagas — key-schema drift is `CF` blocking.

### INTERFACES ↔ ERRORS

- Every `ERR_CODE` in any endpoint's `possible_errors` list is defined in ERRORS § 3 — unresolved codes are `CF` blocking.
- Every `ERR_CODE` defined in ERRORS § 3 appears in at least one endpoint's `possible_errors` list, one saga compensation, or an explicit "internal only" annotation in ERRORS itself — orphan codes are `CF` medium.
- HTTP status declared by the endpoint matches the HTTP status declared by the code — mismatch is `CF` blocking.

### INTERFACES ↔ SECURITY

- Every boundary in INTERFACES § 1 appears in SECURITY § 3 with authority change, validation point, and data classification — missing boundaries are `CF` high.
- The auth scheme in INTERFACES § 3 is cited verbatim by SECURITY § 7 — rewording or paraphrasing is `CF` high (the two artifacts must evolve together, not drift).
- Every endpoint's rate limit in INTERFACES § 6 appears in SECURITY § 11 abuse-prevention summary — silent omission is `CF` medium.

### BEHAVIOR ↔ QUALITY

- Every `SAGA-name` has at least one `METRIC-*` measuring duration or completion count — un-measured sagas are `CF` high.
- Every state-machine transition worth counting has a `METRIC-*` — thin coverage is `CF` medium.
- Tracing spans named in QUALITY § 5 match saga step numbers in BEHAVIOR § 2 — numbering drift (span tagged to step 3 when BEHAVIOR has only 2 steps) is `CF` blocking.
- Event-emission points in BEHAVIOR § 6 match `EVT-*` used in QUALITY § 2 domain log events — drift is `CF` high.

### QUALITY ↔ SECURITY

- Logging forbidden-fields list in QUALITY § 1 covers every sensitive column classified in DATA § 8 and every credential class named in SECURITY § 9 — gaps are `CF` blocking (log leaks).
- Security alerts named in SECURITY (e.g., `ALERT-auth-anomaly`) appear in QUALITY § 6 Alert Catalogue — drift is `CF` high.

### SECURITY ↔ USE_CASES (only if USE_CASES read)

- § 2.1 Authorized actors in SECURITY is a one-to-one map of USE_CASES § 1 Actors — insertions or omissions are `CF` blocking.
- Every `UC-NN` in USE_CASES § 2 has a row in SECURITY § 8 use-case-to-permission map — missing rows are `CF` blocking.

---

## 4. Completeness probes

Apply each sub-section to the design suite with the named downstream consumer in mind. Each gap produces an `MF-NN` finding citing the failing step.

### For continuous per-trigger work (begins after `pass`)

- Every DOMAIN aggregate has enough responsibility detail in DOMAIN § 4 for a per-trigger `/SPEC.md` to scope a unit against it — thin aggregates (name + one line) are `MF` high.
- ARCHITECTURE § 2 names a component per bounded context — components missing for contexts that will need independent deployment are `MF` blocking.
- DOMAIN's bounded-context catalog is explicit and stable — these contexts become the area folders for `units/<area>/u<NN>/`; missing or unclear contexts force the orchestrator to invent area names.
- Every `EP-name` has explicit idempotency, rate-limit, and auth fields — incomplete endpoint spec forces SPEC authors to invent contract details.
- Every `SAGA-name` has step list and compensation — sagas with only a description are `MF` blocking (per-unit SPECs cannot derive their behavioral contract).

### For `/spec` (per-unit)

- Every endpoint's request and response schemas are field-complete — missing fields force `/spec` to invent them.
- Every error code used by a unit's endpoints is defined — unresolved `ERR_*` blocks `/spec` from declaring the error contract.
- Every mutation has an invariant statement in DOMAIN that `/spec` can translate to assertions — missing invariants are `MF` high.
- Every `METRIC-*` has labels and cardinality — `/spec` cannot declare the emission contract without these.
- Every `THREAT-NN` relevant to a unit has named mitigations that the unit will implement — orphan threats block security-aware `/spec`.

### For `/system-verify`

- Every use case has a reachable scenario — orphan UC-NN are `MF` high.
- Every SLO has a measurable SLI — SLOs that cannot be evaluated from the system's emitted metrics are `MF` blocking.
- Every saga has a compensation path that exercises the failure mode — incomplete compensation is `MF` high.
- Every external integration in OPERATIONS § 6 has a failure behaviour (skipped here if OPERATIONS is not read — surface as a Phase 4 finding when QUALITY § 11 Benchmarks references the integration).

---

## 5. Severity calibration heuristics

Severity is defined by downstream impact, not reviewer intensity.

### Blocking

A finding is `blocking` when **any** of the following holds:

- A stable-ID citation does not resolve (`MISSING` in § 5 matrix).
- A DOMAIN invariant has no enforcement path (DATA constraint, BEHAVIOR transition guard, or explicit application-layer annotation).
- A `SAGA-*` has no idempotency key while INTERFACES mandates one.
- A `THREAT-NN` has no `MIT-NN` and no residual-risk row.
- A `SLO-*` has no burn-rate alert.
- A frontmatter count contradicts the body for a required-artifact count.
- A use case (when USE_CASES is read) has no realisation path.

### High

A finding is `high` when the design will confuse downstream readers and force a discretionary fix, but the wrong code can be unwound without cascade regeneration — e.g., an orphan `ERR_CODE` (defined but unused), a thin metric with missing cardinality, an ADR with no "alternatives considered" section.

### Medium

Cosmetic or consistency issue that degrades readability without blocking execution — e.g., component-name spelling drift between ARCHITECTURE and DOMAIN when the field does not propagate to a wire schema.

### Low

Minor wording or style inconsistency — e.g., different capitalisation of the same prose term across two artifacts. Use sparingly; most low-severity items are not worth emitting at all.

### The inflation anti-pattern

If more than 40% of findings in a review are `blocking`, re-examine the severity of each before finalising. Over-inflation means the verdict is always `fix-required`, the orchestrator never exits the loop, and the design phase never completes. Prefer downgrading (with an explicit justification in the finding body) to preserving a subjective blocking rating.

### The under-call anti-pattern

Conversely, a review with zero blocking findings that proposes dozens of medium findings is signalling "this was a vibes check, not a review". If the design suite is materially inconsistent, say so with `blocking`. The quality of the gate depends on the reviewer's willingness to call the verdict honestly.
