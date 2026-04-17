# Per-Intent Blueprint Template

Every intent template in the inventory gets one entry in the Per-Intent Blueprints section of `VOICE_IA.md`, following the template below. Every field is mandatory unless explicitly marked with an "N/A" fallback.

When iterating the inventory in Phase 5 of the workflow, copy the block below per intent and fill in every field. Describe behavior by invocation contract, dialog effects, and response role — not by code shape (Rule 6). Describe responses by role, not by literal spoken string (Rule 7). Reference slot types by name from the Slot Model (Rule 10). Reference context keys from the Dialog Model & Context section (Rule 11). Draw confirmation behavior from the Confirmation Strategy (Rule 12), fallback / handoff from the Fallback & Handoff Strategy (Rule 13), and privacy / safety from the Privacy & Safety Constraints (Rule 14).

---

## Template block (copy per intent)

```markdown
### `{IntentName}` — {short purpose}

- **Kind:** {action | informational | utility | navigational}
- **Purpose:** {one sentence}
- **Audience:** {user types who invoke this intent}
- **Auth requirement:** {none | account-linked | permission-granted: {permission name, e.g., location, reminders, contacts}}
- **Referenced use cases:** {UC-IDs, or explicit classification: utility | navigational | auth-linking}

**Sample utterances** (5–15 representative phrasings; slot placeholders in braces):

- `{utterance with {slot} placeholders}`
- `{utterance}`
- `{utterance}`
  …

**Slots:**

| Name | Type | Required | Prompt role if missing | Confirmation | Validation |
|------|------|----------|------------------------|--------------|------------|
| `{slot}` | `{SlotType}` (from Slot Model) | {yes/no} | {role, e.g., "resource-name slot-elicitation"} | {yes/no} | {validation rule or "none beyond type"} |

(Or "No slots.")

**Confirmation required?** {yes / no — must be consistent with Confirmation Strategy; destructive intents = yes with no exception unless justified inline}

**Primary response role:** {one phrase — e.g., "fork-success confirmation with follow-up suggestion", "resource-status answer with actionable next step"} *(Or "No primary response role — utility intent.")*

**Alternative response variations:** {list of roles — e.g., "condensed role for experienced users (detected via repeat invocation)", "verbose role on first use", "re-prompt role on repeat request". Roles only — not literal strings.}

**Multimodal expansion:**
- **Voice-only devices:** {rule — typically "voice response only"}
- **Voice+screen devices:** {visual card content role, touch controls, or "voice-only — no card"}
- **Companion-push behavior:** {when the dialog hands off to the companion mobile app via push, what payload is passed, or "None."}

(Or "Voice-only — no multimodal expansion." when the app does not target screen devices.)

**Data dependencies:** {what the intent needs from backend — e.g., "user's namespace, resource metadata, fork-permission check". Reference contract-registry endpoint names when present.}

**Context read:** {context keys this intent consumes — e.g., `last_resource` for follow-up resolution, `user_preferences.default_namespace`. Or "None."}

**Context written:** {context keys this intent writes — e.g., `last_fork = {resource_id}` with TTL 5 min. Or "None."}

**Context cleared:** {context keys this intent invalidates — e.g., `last_listing` cleared because topic changed. Or "None."}

**Failure matrix:**

| State | What the agent says (role) | Session | Context written |
|-------|---------------------------|---------|-----------------|
| Fulfilled | {primary response role} | {remain open / end} | {key(s) written} |
| Slot unresolved after re-prompt limit | {fallback role after N tries} | {end} | {none / reset} |
| Slot low-confidence or ambiguous | {disambiguation sub-dialog role} | {remain open} | {disambiguation candidates} |
| Permission required but not granted | {hand-off-to-linking role or consent-request role} | {remain open / end} | {linking-pending flag} |
| Backend error | {generic-failure role with "try again later"} | {end or remain open} | {none} |
| Content filtered (sensitive topic) | {safety response role — decline / disclaim / redirect} | {end or remain open} | {none} |
| Out of scope | {redirect-to-capability role} | {remain open} | {none} |
| Silence / no input | {re-prompt role; after N silences → fallback} | {remain open → end} | {none} |
| Session timeout | {implicit end-of-session; no speech} | {end} | {session-state cleared} |
| Confirmation declined | {acknowledgment + return-to-prior-state role or end} | {remain open / end} | {clear confirmation-pending flag} |

(Use "N/A — {reason}" for rows that genuinely do not apply — e.g., confirmation-declined on an intent that never confirms, content-filtered on an intent whose utterances cannot trigger a sensitive topic.)

**Handoff paths:** {which handoff targets this intent can route to — names from Fallback & Handoff Strategy — e.g., "human-operator on repeated failure; companion-mobile-app for complex branching". Or "None."}

**Platform variations:** {per-platform differences — e.g., "Alexa: uses Alexa.Reminders API for follow-up; Google: uses notification channel; Siri: exposed as App Intent with parameter set X". Or "identical across platforms". Or "single-platform — no variations."}

**Session end after fulfillment?** {session remains open for follow-up (with explicit re-prompt role) | session closes immediately | conditional — rule}

**Shared surfaces invoked:** {names from Shared Surfaces section — e.g., "confirmation-prompt, account-linking-prompt". Or "None."}
```

Repeat the block for every intent template in the inventory.

---

## Field-by-field guidance

- **Intent name** — `PascalCase`, matching the platform's intent-naming convention (Alexa intents, Google intent names, Siri App Intent type names). Use realistic names that reflect the user's goal (`ForkResource`, `GetStatus`, `StartCheckout`), not abstract placeholders.
- **Kind** — *action* mutates or fetches data; *informational* answers a factual or status question without mutation; *utility* is `help`, `stop`, `cancel`, `repeat`, `launch`; *navigational* moves between flows or sections within a dialog without its own primary outcome.
- **Auth requirement** — "none" means the intent works without account linking or device permissions; "account-linked" requires the user has linked their app account; "permission-granted" requires a specific device permission (location, reminders, contacts) — name the permission.
- **Sample utterances** — 5–15 distinct representative phrasings covering the common ways users would express the intent. Use slot placeholders in braces (`{resource}`, `{date}`). Cover short-form, long-form, and one-shot-with-slots phrasings. Do not duplicate phrasings that differ only in a synonym already handled by the slot type's synonym list.
- **Slots** — one row per slot. Reference the slot type by name from the Slot Model — do not re-enumerate canonical values or synonyms (Rule 10). Prompt role describes the agent's re-prompt when the slot is missing — a role, not a literal string. Confirmation column indicates whether this specific slot requires explicit per-slot confirmation (separate from intent-level confirmation).
- **Confirmation required?** — yes or no. Must be consistent with the Confirmation Strategy (Rule 12). A destructive intent (delete, send, pay, publish, overwrite) with `Confirmation required? no` requires an inline justification; otherwise it is a rule violation.
- **Primary response role** — one role describing the success response. Analog of the primary CTA in web IA. Every non-utility intent declares exactly one primary response role (Rule 2). If an intent legitimately has two roles, split it or justify the pairing inline.
- **Alternative response variations** — response roles that vary by context (first-use vs repeat, verbose vs condensed, screen-present vs voice-only). Roles only — not literal strings (Rule 7).
- **Multimodal expansion** — if the app targets screen devices, declare the visual-card content role, touch controls, and companion-push behavior. If voice-only, state "Voice-only — no multimodal expansion." Reference the Multimodal Expansion section's conventions.
- **Data dependencies** — high-level, not implementation. Reference contract-registry endpoint names if a CONTRACT_REGISTRY.md exists; do not inline wire formats.
- **Context read / written / cleared** — use the context keys declared in the Dialog Model & Context section (Rule 11). Include TTLs inline when they differ from the section default.
- **Failure matrix** — every row from the template. Use "N/A — {reason}" for genuinely inapplicable rows. Session column is *remain open* (re-prompt expected), *end* (clean close), or conditional. Context column lists keys written or cleared at this state.
- **Handoff paths** — reference targets from the Fallback & Handoff Strategy (Rule 13) by name. Do not redefine the handoff flow.
- **Platform variations** — the key field for multi-platform apps (Rule 18). When behavior is identical, say so explicitly; do not leave the field blank. When divergent, enumerate per-platform. For single-platform apps, "single-platform — no variations" is the correct value.
- **Session end after fulfillment?** — voice UX varies: a one-shot `GetStatus` typically closes the session; a mid-flow `AddItem` typically keeps the session open for follow-ups. The rule can be conditional ("remain open if used inside `CheckoutFlow`; end otherwise").
- **Shared surfaces invoked** — reference by name from the Shared Surfaces section (Rule 16). Do not redescribe the surface.

---

## Common mistakes to avoid

- Creating a separate intent per sample-utterance phrasing instead of consolidating variants under one intent template (Rule 1). Synonyms handled by the slot type are not separate intents; phrasings like `fork {resource}` vs `make a copy of {resource}` vs `clone {resource}` belong to the same intent.
- Writing handler-class or SDK-specific code ("`canHandle` checks `Alexa.getIntentName(…)`"). Those belong in per-unit SPECs (Rule 6).
- Re-enumerating canonical values or synonyms in an intent's slot table — reference the Slot Model type by name only (Rule 10).
- Redefining session-scope or context TTLs in an intent entry — reference the Dialog Model & Context section (Rule 11).
- Declaring `Confirmation required? no` on a destructive intent without inline justification (Rule 12).
- Writing literal spoken copy ("Great — I forked it for you.") instead of a role ("fork-success confirmation with follow-up suggestion") (Rule 7).
- Leaving a failure-matrix row as "TBD" or omitting rows. Use "N/A — {reason}" when genuinely inapplicable.
- Describing earcons, TTS voices, or speaking rates (Rule 8). Audio execution is owned downstream.
- Omitting platform variations when multiple platforms are targeted (Rule 18). Either enumerate divergence or explicitly state "identical across platforms".
- Redescribing a shared surface (e.g., confirmation-prompt, account-linking-prompt) inline per intent (Rule 16). Reference it by name.
- Silently dropping a use case — every use case must appear in the traceability matrix, either mapped to intents or noted "no voice surface" with a reason (Rule 5).
