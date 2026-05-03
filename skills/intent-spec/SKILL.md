---
name: INTENT_SPEC.md
description: Produce a deep, per-intent design specification for one voice intent — full sample-utterance library, slot resolution detail, turn-by-turn dialog flow, confirmation flow detail, response composition with SSML hints, multimodal expansion detail, expanded failure matrix, context effects, and intent-specific privacy and safety constraints. All citations reference VOICE_IA's stable IDs (intent IDs, slot names, response role names, context keys, failure-matrix rows) and DESIGN.md tokens (tone register, persona, SSML hints) verbatim. Use when asked to create an INTENT_SPEC.md for a voice intent, deepen the per-intent blueprint into a full dialog specification, design multi-turn slot-filling for an intent, specify confirmation flow detail, or produce an INTENT_SPEC.md.
---

# Task: Create INTENT_SPEC.md for a Voice Intent

## Objective

Produce an INTENT_SPEC.md that serves as the single source of truth for designing the deep dialog of one voice intent: a complete sample-utterance library grouped by phrasing pattern, every slot's validation / disambiguation / re-prompt / carry-over / confirmation rules, the turn-by-turn multi-turn dialog flow, the confirmation flow detail, every declared response role's composition (TTS body structure, SSML hints, variation strategy, length budget, persona constraints), the multimodal expansion detail, an expanded failure matrix per failure type, the intent's full context-effect ledger, and intent-specific privacy and safety constraints. The defining discipline — and the definition of "done" — is that **the gap between VOICE_IA's blueprint and a fully-composable dialog is purely mechanical**: an implementing agent reading VOICE_IA's blueprint for the intent + DESIGN.md's voice-design tokens + this INTENT_SPEC produces the intent's full dialog, slot-filling logic, confirmation pattern, response structure, and multimodal expansion exactly as designed, without inventing utterance grammar, disambiguation prompts, or escape paths.

The commonest violation is silently picking an answer for a design-level question instead of surfacing it. When VOICE_IA, DESIGN.md, or sibling INTENT_SPECs are silent, ambiguous, or contradictory on a question this intent must answer — a re-prompt sequence, an SSML hint, an escape path, a multimodal action affordance — the question belongs in § 11 Open Questions with options, tradeoffs, and a recommendation. Do not invent. INTENT_SPEC must never introduce a design decision that is not already traceable to VOICE_IA, DESIGN.md, or a sibling INTENT_SPEC.

INTENT_SPEC is **opt-in**. Most intents stay with their lightweight inline blueprint in VOICE_IA. Use INTENT_SPEC only when the intent meets at least one of the selection criteria below.

### Selection criteria — when to write an INTENT_SPEC

- The intent's full composition exceeds ~30 lines if expressed inline in VOICE_IA.
- The intent is multi-turn — needs more than one user-system exchange to complete.
- Slot-filling involves disambiguation, re-prompts, or carry-over from prior turns.
- Confirmation flow is non-trivial — multi-stage, escape paths, ambiguous affirmations.
- Multimodal expansion is non-trivial — visual card with multiple action affordances.
- Failure paths include hand-off to a human.
- Platform variations are substantial — different dialog flows on Alexa vs Google.
- Multiple work units touch the same intent.
- The intent is high-traffic or high-stakes — financial, medical, destructive.

Intents not meeting these stay with inline blueprints in VOICE_IA. Do not write an INTENT_SPEC for a one-shot informational intent or a utility intent.

The output file is conventionally placed at `voice-intents/<intent-id>/INTENT_SPEC.md` in the consuming project, where `<intent-id>` is the stable identifier from VOICE_IA's Intent Inventory.

---

## Inputs

1. **Intent identifier** (required) — the stable intent ID from VOICE_IA's Intent Inventory (e.g., `ForkResource`, `StartCheckout`). Used verbatim in the frontmatter `intent_id` field and in the title of the output.
2. **VOICE_IA.md** (required) — invocation model, slot model, dialog model and context, platform strategy, confirmation strategy, fallback and handoff strategy, privacy and safety constraints, and the intent's lightweight blueprint to extend. The authoritative source for intent IDs, slot names, response role names, context keys, shared-surface names, failure-matrix row vocabulary, handoff target names, and platform names. INTENT_SPEC extends — never redefines — VOICE_IA's claims about this intent.
3. **DESIGN.md** (required if present) — voice-style tokens: tone-of-voice register (`tone-warm`, `tone-formal`, `tone-playful`), persona descriptors (`persona-helpful`, `persona-expert`), prosody guidance, SSML conventions (named hints — `pause-short`, `emphasis-moderate`, `prosody-deliberate`), multimodal card style tokens, microcopy patterns. INTENT_SPEC cites these tokens by name and never invents new ones. If DESIGN.md is absent, surface the absence as an open question — do not invent voice-style decisions.
4. **Sibling INTENT_SPEC.md files** (optional, auto-discovered) — the INTENT_SPECs of every intent referenced from this intent's dialog flows (intents this hands off to, or receives a hand-off from). Discovery mechanism: walk the dialog flow's hand-off targets and load the corresponding `voice-intents/<intent-id>/INTENT_SPEC.md` for each; if no INTENT_SPEC exists for a referenced intent, it is sufficient to cite VOICE_IA's blueprint.
5. **Prior `voice-intents/<intent-id>/INTENT_SPEC.md`** (optional, auto-discovered — only if refining) — read fully; preserve every resolved open question, every utterance pattern already calibrated, every re-prompt sequence already approved. Re-run citations against the current VOICE_IA and DESIGN.md — do not assume stale intent IDs, slot names, response role names, or SSML hint names are still valid.

Read-set size: VOICE_IA + DESIGN.md are always read end-to-end; 0–4 sibling INTENT_SPECs depending on hand-off targets in the dialog flow. Read every sibling INTENT_SPEC in full — truncated reads cause silent contradiction across hand-offs.

---

## Rules

### Citation discipline

INTENT_SPEC is the bridge between VOICE_IA's intent blueprint, DESIGN.md's voice tokens, and per-unit voice SPECs. Every claim that resolves to another document must cite it by stable name — never by prose, never by re-stated content.

- **Intent IDs** — cite VOICE_IA's intent ID verbatim (e.g., `ForkResource`). Use the same casing, the same name. Do not invent variants.
- **Slot names and types** — cite VOICE_IA's Slot Model entries by name (e.g., `resource` slot of type `ResourceName`). Do not redefine canonical values, synonyms, or resolution rules — those are VOICE_IA's.
- **Response role names** — cite VOICE_IA's per-intent blueprint role names verbatim (e.g., "fork-success confirmation with follow-up suggestion", "not-found clarification"). INTENT_SPEC composes the role's TTS structure; it does not rename or re-classify roles.
- **Context keys** — cite VOICE_IA's Dialog Model & Context section keys verbatim, including their TTLs. If this intent needs a context key VOICE_IA has not declared, surface it in § 11 — do not invent the key here.
- **Shared-surface names** — cite VOICE_IA's Shared Surfaces section entries by name (e.g., `confirmation-prompt`, `account-linking-prompt`). Do not redescribe the surface — reference it.
- **Handoff target names** — cite VOICE_IA's Fallback & Handoff Strategy targets by name (e.g., `human-operator`, `companion-mobile-app`).
- **Platform names** — use canonical names (Alexa, Google, Siri, custom) — match VOICE_IA's Platform Strategy spelling.
- **Failure-matrix rows** — use VOICE_IA's row vocabulary (slot-not-filled, ambiguous-slot, NLU-confidence-low, intent-out-of-scope, downstream-error, network-timeout, permission-denied, account-not-linked, content-unavailable, silence, session-timeout, confirmation-declined). The expanded matrix in § 8 deepens these rows; it does not rename them.
- **DESIGN.md tokens** — cite tone register, persona, and SSML hints by their token name (`tone-warm`, `persona-helpful`, `pause-short`, `emphasis-moderate`). Do not invent SSML hint names. If a hint is needed that DESIGN.md has not declared, surface it in § 11.
- **Sibling intents** — cite by VOICE_IA intent ID. When the dialog flow hands off to or receives from another intent, name it by its VOICE_IA ID, not by a description.

### No overlap with VOICE_IA

INTENT_SPEC extends VOICE_IA's blueprint — it does not duplicate or contradict it. Any field VOICE_IA owns must not be redefined here. Specifically forbidden:

- Redefining sample utterance counts, the slot list, the primary response role name, the failure matrix summary, the context model, the platform strategy, the confirmation strategy, the fallback and handoff strategy, or the privacy and safety constraints. Those are VOICE_IA's.
- Defining new slot types or response roles — VOICE_IA owns those. If the intent legitimately needs a new slot type or response role, surface it in § 11 and re-run VOICE_IA before regenerating this INTENT_SPEC.
- Brand-voice decisions — DESIGN.md owns persona, tone register, and prosody guidance. INTENT_SPEC cites these tokens; it does not author them.

### No implementation detail

Do not write handler class structure, SDK-specific invocations, NLU model internals, ASR confidence-threshold code, backend integration code, or library names. Specifically forbidden:

- No `Alexa Skills Kit`, `Dialogflow`, `App Intents`, `Rasa`, or other library names. INTENT_SPEC describes the dialog; per-unit SPECs map it to a stack.
- No handler class methods (`canHandle`, `handle`, `getSlotValue`).
- No NLU model thresholds expressed as numeric code (`if confidence < 0.7`). State the policy ("low confidence triggers disambiguation"); the threshold value belongs in VOICE_IA's Slot Model or in DESIGN.md, not here.

### No final TTS strings

INTENT_SPEC describes response **structure and constraints**, not literal copy. Specifically forbidden:

- No exact spoken words ("Great — I forked it for you. Want to open it?").
- No final prompt strings.
- One short placeholder phrase per slot-elicitation or disambiguation example is acceptable as illustrative only — never as the canonical string. Final voice copy is a downstream UX-writing step.

What INTENT_SPEC owns instead: the response role's body structure (opening / core / closing), the SSML hint sequence (cited by name from DESIGN.md), the variation strategy (cycle length cap, what may vary), the length budget (target word counts for short / medium / long variants), and the persona constraints (what the persona must / must not say, cited by token from DESIGN.md).

### Completeness

- Every section of the output template is mandatory. Do not omit § 7 Multimodal Expansion when the intent is voice-only — state "Voice-only intent; no multimodal expansion." instead.
- Every slot listed in VOICE_IA's per-intent blueprint must have a § 3 entry. Slot-detail counts must match VOICE_IA's slot list for this intent.
- Every response role declared in VOICE_IA's per-intent blueprint (primary plus alternative variations) must have a § 6 entry. Response-role counts in the frontmatter match the count of § 6 entries.
- Every failure type listed in the rules below must appear in § 8, either with full detail or with `N/A — {reason}`. Do not silently drop rows.
- Every context key VOICE_IA declares for this intent must appear in § 9 under read / written / cleared, or be explicitly absent with a one-line reason.
- The Open Questions section reads "All questions resolved." only when no questions remain. Otherwise, status is `has_open_questions` and § 11 lists every unresolved item.

### Single YAML frontmatter block

- Exactly one YAML frontmatter block at the top of the INTENT_SPEC.md output, never two — even when refining an existing INTENT_SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `intent_id`, `platforms`, `utterance_patterns`, `slots_detailed`, `dialog_turns_specified`, `response_roles_specified`, `failure_modes_specified`, `multimodal`, `open_questions`).
- Every count in the frontmatter matches the body exactly: `utterance_patterns` is the number of phrasing-pattern subsections in § 2; `slots_detailed` is the number of slot entries in § 3; `dialog_turns_specified` is the number of turn entries in § 4; `response_roles_specified` is the number of response-role entries in § 6; `failure_modes_specified` is the number of failure-type entries in § 8 with full detail (rows marked `N/A — {reason}` are not counted); `open_questions` is the number of unresolved questions in § 11 (zero when § 11 reads "All questions resolved.").
- `multimodal` is `true` when § 7 contains substantive content; `false` when § 7 reads "Voice-only intent; no multimodal expansion."
- `platforms` is the subset of `{alexa, google, siri, custom}` from VOICE_IA's Platform Strategy that this intent targets.
- `status` is `complete` only when § 11 reads "All questions resolved."; `has_open_questions` when one or more questions remain unresolved; `blocked` only when a missing input (e.g., missing DESIGN.md, missing VOICE_IA blueprint for this intent) prevents authoring INTENT_SPEC and the gap is explicit in § 11.

### Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact intent IDs, slot names, response role names, context keys, SSML hint names, and platform names. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

The INTENT_SPEC.md file must follow this exact structure. Every section is mandatory. If a section has no content for a legitimate reason (e.g., no confirmation required), include the heading with the explicit "no-content" sentence under it (e.g., "No confirmation required for this intent. Rationale: {reason from VOICE_IA's Confirmation Strategy}.") — do not omit headings.

```markdown
---
skill: INTENT_SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
intent_id: {intent-id-from-VOICE_IA}
platforms: [{alexa, google, siri, custom} — subset]
utterance_patterns: {N}
slots_detailed: {N}
dialog_turns_specified: {N}
response_roles_specified: {N}
failure_modes_specified: {N}
multimodal: {true | false}
open_questions: {N}
---

# INTENT_SPEC: `{IntentID}` — {short purpose}

## 1. Identity & Cross-references

- **Intent ID:** `{IntentID}` (from VOICE_IA Intent Inventory)
- **Trigger nature:** {top-level launch | contextual | follow-up only}
- **Platforms targeted:** {subset of {Alexa, Google, Siri, custom} from VOICE_IA's Platform Strategy}
- **Tone register:** `{tone-token}` (from DESIGN.md)
- **Persona:** `{persona-token}` (from DESIGN.md)
- **Sibling intents:** `{IntentID}`, `{IntentID}` (intents this hands off to or receives from in § 4 dialog flow). If none: "None — self-contained intent."

## 2. Sample Utterance Library

A complete enumerated set of sample utterances grouped by phrasing pattern. Each pattern is its own subsection.

### Pattern: {one-line label, e.g., "imperative direct"}

- **Pattern shape:** `{template with {slot} placeholders}`
- **Concrete examples** (5–15):
  - `{example 1}`
  - `{example 2}`
  - …
- **Slot-filled vs slot-elided variants:** {describe which slots may be elided in this pattern; how the dialog re-prompts}
- **Edge phrasings:** {interruptions, partial utterances, hesitation tokens — `{example with "uh"}`, `{example trailing off}`. Or "None — pattern admits no edge phrasings."}

(Repeat per phrasing pattern. If the intent has only one pattern, list it.)

## 3. Slot Detail

For each slot in this intent (from VOICE_IA's per-intent slot list):

### Slot: `{slot}` of type `{SlotType}`

- **Validation rule:** {what makes a value acceptable — e.g., "value must be a resource the caller owns or a public resource"}
- **Disambiguation prompt:** {response role from VOICE_IA used when ambiguous — e.g., "disambiguation sub-dialog asking the user to pick from a numbered list"}
- **Re-prompt sequence:**
  1. **First re-prompt:** {role}
  2. **Second re-prompt:** {role}
  3. **Escalation:** {role — typically fallback or hand-off}
- **Default fill rules:** {when the slot can be inferred from context — cite context keys; or "No defaults — slot must be supplied or elicited."}
- **Carry-over from prior turns:** {when a value persists across turns; expiry rule citing context-key TTL from VOICE_IA. Or "No carry-over."}
- **Confirmation rule:** {implicit | explicit | none — must be consistent with VOICE_IA's per-slot confirmation column}

(Repeat per slot. If the intent has no slots: "No slots for this intent.")

## 4. Dialog Flow

A turn-by-turn specification of the multi-turn dialog. Choose the format that is clearer for this intent — state-transition table or numbered turn-by-turn narrative — and use it consistently.

### Format A — state-transition table (preferred for branching dialogs)

| Turn | Trigger condition | System action | Context read | Context written / cleared | Exit conditions |
|------|-------------------|---------------|--------------|---------------------------|-----------------|
| {N} | {state + user input that leads here} | {slot-fill / confirm / answer / hand-off / re-prompt} | {keys} | {keys with TTL or "cleared"} | {what user inputs end the dialog — success / failure / escape} |

### Format B — numbered turn-by-turn narrative (preferred for linear dialogs)

1. **Turn 1 — {label}:** {trigger condition}. System {action}. Reads {context keys}. Writes {context keys with TTL}. Exits to turn 2 on {condition}; exits to {success / failure / escape} on {condition}.
2. **Turn 2 — {label}:** …

(Render whichever format is clearer. Every turn has trigger, action, state transition, and exit conditions.)

## 5. Confirmation Detail

(If `Confirmation required? = no` in VOICE_IA's blueprint for this intent: "No confirmation required for this intent. Rationale: {reason from VOICE_IA's Confirmation Strategy}." — and skip the rest of this section.)

- **When confirmation is required:** {trigger conditions — destructive action, high-stakes commitment, ambiguous slot value above a threshold}
- **Confirmation prompt structure:** {what is repeated back — target entity, action verb, consequence summary; what the affirmation pattern is}
- **Affirmative grammar:** {accepted forms — yes / yeah / sure / confirm / go ahead / do it. Cite VOICE_IA's Confirmation Strategy if standardized.}
- **Negative grammar:** {accepted forms — no / nope / cancel / never mind / stop. Cite VOICE_IA's Confirmation Strategy if standardized.}
- **Escape path:** {how the user backs out without confirming or denying — e.g., "the user says 'help' or 'go back' → re-enter slot-elicitation for the disputed slot; the user says 'stop' → end session via session-closing surface"}
- **Timeout behavior:** {what the system does on silence — re-prompt with `confirmation-prompt` shared surface; after N silences, treat as decline; cite re-prompt limit from VOICE_IA's Fallback strategy}

## 6. Response Composition

For each response role declared in VOICE_IA's per-intent blueprint (primary plus every alternative variation):

### Role: "{role name from VOICE_IA — verbatim}"

- **TTS body structure:**
  - **Opening:** {what the opening conveys — acknowledgment, transition, status preamble}
  - **Core content:** {what the body conveys — the answer, the confirmation, the next-step suggestion}
  - **Closing:** {what the closing conveys — re-prompt, session-end signal, follow-up invitation. Or "No closing — implicit session continuation."}
- **SSML hints:** {sequence of named hints from DESIGN.md — e.g., "`pause-short` between opening and core; `emphasis-moderate` on the resource name; `prosody-deliberate` on the consequence summary". Do not invent hint names.}
- **Variation strategy:** {randomized phrasings to avoid repetition; cycle length cap — e.g., "minimum cycle of 3 phrasings before any repeats within a session"; what may vary (opening verb, closing follow-up); what must remain stable (resource name, consequence summary)}
- **Length budget:**
  - **Short:** ~{N} words (used in {context — e.g., "repeat-user, hands-busy"})
  - **Medium:** ~{N} words (default)
  - **Long:** ~{N} words (used in {context — e.g., "first-use, screen-present"})
- **Persona constraints:** {what the persona `{persona-token}` must / must not say in this role — cite DESIGN.md persona descriptors}

(Repeat per response role.)

## 7. Multimodal Expansion

(If the intent does not target screen-equipped devices: "Voice-only intent; no multimodal expansion." — and skip the rest of this section.)

- **Visual card structure:** {title role, subtitle role, primary text role, image position (leading / trailing / hero / none), action affordances (count and roles)}
- **Touch / tap targets:** {what users can interact with on the card — primary action tap, secondary action tap, dismiss}
- **Companion app expansion:** {when a screen-less response triggers a follow-up on a paired device — push-notification payload role, deep-link target. Or "No companion-app expansion."}
- **Voice-screen synchronization:** {what the screen says vs what TTS says — same-content / TTS is summary / screen is summary / divergent-roles}
- **Card lifetime:** {after speech | on next turn | on timeout — citing VOICE_IA's Multimodal Expansion conventions}

## 8. Failure Matrix Detail

Beyond VOICE_IA's failure-matrix summary for this intent, for each applicable failure type:

### Failure: {failure-type name from VOICE_IA's row vocabulary}

- **Detection signal:** {what makes this failure detectable — slot resolved with confidence below threshold, downstream returns 5xx, no permission grant, etc.}
- **User-perceived behavior:** {what they hear / see — response role and any visual/screen effect}
- **Re-prompt vs fallback vs handoff:** {which response role is used — cite VOICE_IA's Fallback & Handoff Strategy targets and shared-surface names}
- **Retry budget:** {N attempts before escalation; cite VOICE_IA's re-prompt limit}
- **Logging / privacy notes:** {what is captured for diagnostics; what must not be — cite VOICE_IA's Privacy & Safety Constraints}

(Repeat per applicable failure type. Failure types from VOICE_IA's vocabulary: slot-not-filled, ambiguous-slot, NLU-confidence-low, intent-out-of-scope, downstream-error, network-timeout, permission-denied, account-not-linked, content-unavailable, silence, session-timeout, confirmation-declined. Use `N/A — {reason}` for rows that genuinely do not apply — e.g., confirmation-declined on an intent that never confirms.)

## 9. Context Effects

- **Read keys:** {context keys this intent reads — cite VOICE_IA's Dialog Model & Context section keys}
- **Written keys:** {context keys this intent sets, with TTL from VOICE_IA — e.g., `last_fork = {resource_id}` (TTL 5 min)}
- **Cleared keys:** {context keys this intent invalidates — cite VOICE_IA keys; rationale per key — "topic-change", "session-end", "explicit reset"}
- **Cross-intent dependencies:** {which intents must have run before for this one to have its expected context — cite by VOICE_IA intent ID. Or "None — intent is self-bootstrapping."}

## 10. Privacy & Safety

Intent-specific constraints beyond VOICE_IA's global Privacy & Safety:

- **Information not to surface:** {data categories this intent must not voice even when asked — e.g., "never voice the resource SHA in raw form; voice the short-form ID instead". Or "No intent-specific suppressions."}
- **Sensitive data handling:** {masking in logs — which slots are redacted; opt-in confirmations — when the user must explicitly grant access. Or "Inherits global rules from VOICE_IA's Privacy & Safety Constraints."}
- **Account-linking requirement:** {required | optional | not applicable — must match VOICE_IA's Auth requirement for this intent}
- **Age / vulnerability gating:** {if applicable — adult-auth gate, simplified-language mode, etc. Or "None."}
- **Hand-off to human:** {when forced — e.g., "any utterance matching the medical-content pattern routes to `human-operator` via `handoff-prompt`". Or "None — never forced."}

## 11. Open Questions

This section must be EMPTY before the intent is implemented. If any questions remain unresolved, the spec is not ready. The user will resolve open questions after reviewing INTENT_SPEC — do not ask during generation.

When drafting INTENT_SPEC, you will encounter decisions where multiple approaches are defensible and VOICE_IA / DESIGN.md / sibling INTENT_SPECs do not clearly favor one. Do NOT silently pick an answer to keep this section empty. Instead:

- If a question has one obviously correct answer given the input documents, resolve it yourself and write the decision into the appropriate section. Do not list it here.
- If a question has no clear answer — multiple valid approaches exist, or the input documents are ambiguous or silent — list it here with proposed options and tradeoffs so the user can make an informed decision.

Format for open questions:

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {if you lean one way, say so and why — but leave the decision to the user}

Once the user resolves all questions, move each resolution into the appropriate section and replace this section with:

"All questions resolved."
```

---

## Scope

### In scope

- Full per-intent dialog specification for one intent: utterance library, slot resolution, dialog flow, confirmation detail, response composition, multimodal expansion, failure matrix, context effects, privacy and safety
- Citation discipline against VOICE_IA's stable IDs (intent ID, slot names, response role names, context keys, shared-surface names, handoff targets, platform names, failure-matrix rows)
- Citation discipline against DESIGN.md's voice tokens (tone register, persona, SSML hints)
- Selection-criteria gate that keeps INTENT_SPEC reserved for intents that genuinely warrant deep design

### Out of scope

- Defining new slot types, response roles, context keys, shared surfaces, handoff targets, platforms, or failure-matrix rows — owned by `VOICE_IA.md`
- Brand-voice authoring — tone register, persona descriptors, prosody guidance, SSML hint definitions — owned by `DESIGN.md`
- Final literal TTS strings — exact spoken words, prompts, error messages, personality, humor — owned by a downstream voice-writing / UX-writing phase
- Audio aesthetics — specific TTS voice selection, speaking rate, earcon sounds, jingles, stings — owned downstream
- Implementation detail — handler classes, SDK invocations (`Alexa Skills Kit`, `Dialogflow`, App Intents), NLU model internals, ASR thresholds in code, backend integration code — owned by per-unit `SPEC.md`
- Backend / API wire formats — owned by `INTERFACES.md`
- Database schema, business logic, algorithm design — owned by architecture and per-unit SPECs
- Use-case ↔ intent traceability — owned by `VOICE_IA.md`
- Other intents' deep design — each intent gets its own INTENT_SPEC at `voice-intents/<intent-id>/INTENT_SPEC.md`

---

## Quality Checklist

Before considering an INTENT_SPEC.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `intent_id`, `platforms`, `utterance_patterns`, `slots_detailed`, `dialog_turns_specified`, `response_roles_specified`, `failure_modes_specified`, `multimodal`, `open_questions`)
- [ ] Frontmatter counts match the body exactly: `utterance_patterns` = subsections in § 2, `slots_detailed` = entries in § 3, `dialog_turns_specified` = turns in § 4, `response_roles_specified` = entries in § 6, `failure_modes_specified` = full failure entries in § 8 (rows marked `N/A — {reason}` are not counted), `open_questions` = unresolved items in § 11
- [ ] Frontmatter `intent_id` matches an entry in VOICE_IA's Intent Inventory verbatim
- [ ] Frontmatter `platforms` is a subset of VOICE_IA's Platform Strategy targets for this intent
- [ ] Frontmatter `multimodal` is `true` iff § 7 contains substantive content; `false` iff § 7 reads "Voice-only intent; no multimodal expansion."
- [ ] `status` is `complete` iff § 11 reads "All questions resolved."; `has_open_questions` otherwise; `blocked` only when a missing input prevents authoring and § 11 makes the gap explicit
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files (citations are by name; the reader can resolve them by opening VOICE_IA / DESIGN.md)
- [ ] Selection criteria are met — the intent satisfies at least one of: composition exceeds ~30 inline lines, multi-turn, slot-disambiguation, non-trivial confirmation flow, non-trivial multimodal expansion, hand-off to human, substantial platform variation, multiple work units, high-traffic / high-stakes
- [ ] § 1 cites VOICE_IA intent ID, trigger nature (top-level / contextual / follow-up), platforms targeted, tone register token, persona token, and sibling intents
- [ ] § 2 has at least one phrasing pattern with 5–15 concrete examples per pattern, slot-filled vs slot-elided variants noted, edge phrasings either listed or explicitly absent
- [ ] § 3 has one entry per slot in VOICE_IA's per-intent slot list — slot count matches; every entry covers validation, disambiguation prompt (citing a VOICE_IA role), re-prompt sequence (3 stages), default fill, carry-over, per-slot confirmation rule
- [ ] § 4 specifies every turn with trigger condition, system action, state transition (context read / written / cleared), and exit conditions; uses either a state-transition table or numbered narrative — not both
- [ ] § 5 either describes the confirmation flow with prompt structure, affirmative grammar, negative grammar, escape path, timeout behavior — or states "No confirmation required for this intent. Rationale: {reason}." consistent with VOICE_IA's Confirmation Strategy
- [ ] § 6 has one entry per response role declared in VOICE_IA's per-intent blueprint — every entry covers TTS body structure (opening / core / closing), SSML hints (cited from DESIGN.md), variation strategy with cycle cap, length budget (short / medium / long), persona constraints (cited from DESIGN.md)
- [ ] § 7 either describes multimodal expansion (visual card structure, touch targets, companion app, voice-screen sync, card lifetime) or states "Voice-only intent; no multimodal expansion." — never partial
- [ ] § 8 lists every applicable failure type from VOICE_IA's row vocabulary; rows marked `N/A — {reason}` are explicit, not silently dropped; full entries cover detection signal, user-perceived behavior, re-prompt vs fallback vs handoff (citing VOICE_IA targets), retry budget, logging / privacy notes
- [ ] § 9 cites only context keys declared in VOICE_IA's Dialog Model & Context section; TTLs match VOICE_IA's section; cross-intent dependencies cite intents by VOICE_IA ID
- [ ] § 10 names intent-specific privacy / safety constraints distinct from VOICE_IA's global rules, or explicitly inherits them; account-linking requirement matches VOICE_IA's Auth requirement for this intent; hand-off-to-human conditions cite VOICE_IA targets
- [ ] No new slot types, response roles, context keys, shared surfaces, handoff targets, platforms, or failure-matrix rows are introduced — every such name traces to VOICE_IA verbatim
- [ ] No new SSML hints, tone tokens, or persona tokens are introduced — every such name traces to DESIGN.md verbatim
- [ ] No final literal spoken copy — response composition describes structure and constraints; at most one short placeholder phrase per slot-elicitation or disambiguation example
- [ ] No implementation library names (no `Alexa Skills Kit`, `Dialogflow`, `App Intents`, `Rasa`); no handler-class methods; no NLU thresholds expressed as code
- [ ] No audio aesthetics — no specific TTS voice selection, pitch settings, speaking rates as numbers, earcon sound choices, jingle content
- [ ] If the intent's hand-off targets include sibling intents with their own INTENT_SPEC, those are cited by VOICE_IA ID and the hand-off semantics are consistent across both INTENT_SPECs
- [ ] Every "no-content" section ("No confirmation required…", "Voice-only intent…", "No slots…") uses the explicit no-content sentence — never a silent omission
