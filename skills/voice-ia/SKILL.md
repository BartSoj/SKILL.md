---
name: VOICE_IA.md
description: Generate a voice information architecture registry — invocation model, slot model, dialog model and context, intent inventory, per-intent blueprints, multi-turn flows, shared surfaces, confirmation strategy, fallback and handoff, privacy and safety, platform strategy, and use-case traceability. Use when asked to create a voice IA, design a voice skill or voice agent, produce an intent inventory, define the voice dialog model, document sample utterances and slots, or produce a VOICE_IA.md.
---

# Task: Generate VOICE_IA.md — Voice Information Architecture Registry

## Objective

Produce a VOICE_IA.md that serves as the single source of truth for a voice-driven conversational application's information architecture: the invocation model (wake word, invocation name, launch intents, deep-invocation phrases), the full intent inventory, per-intent blueprint (purpose, sample utterances, slots, primary response role, confirmation rules, failure matrix, context effects, multimodal expansion), multi-turn dialog flows, shared surfaces (re-prompt, fallback, disambiguation, confirmation, handoff, linking), context model (what is remembered within and across sessions), platform strategy (Alexa / Google / Siri / custom), multimodal expansion strategy, account-linking strategy, privacy and safety constraints, and bidirectional traceability from product use cases to intents. An agent or designer reading this document knows exactly what intents exist, how users invoke them, what slots they consume, how the dialog carries state across turns, how each intent fails, and how every use case is realized on the voice surface — without making any independent grammar, naming, or confirmation decisions. VOICE_IA sits between product requirements (proposal, use cases) and per-unit voice SPECs, the same way WEB_IA sits between product requirements and per-unit web SPECs.

This skill covers **voice-first conversational interfaces** — surfaces where voice input is the primary modality and voice output is a primary modality. It does not cover text chatbots (a future `CHAT_IA.md` would handle those) unless the chatbot shares the same intent model; in that case, scope it in explicitly in Phase 1.

---

## Inputs

1. **Product / proposal document** (required) — product purpose, target users, and rationale. Sets the scope and audience frame for the IA.
2. **Use cases document** (required) — enumerates user interactions and jobs-to-be-done. Primary input. Every use case with a voice surface must map to one or more intents in the inventory.
3. **Architecture document** (optional but recommended) — target voice platforms (Alexa Skill, Google Action, Siri App Intents, custom stack with ASR / NLU / TTS), dialog framework, backend endpoints the voice app hits, account-linking model. Informs the platform strategy, slot model, and fallback design. If absent, infer conservative defaults and flag non-obvious choices as open questions.
4. **Existing voice codebase** (auto-discovered) — discover current intents, sample utterances, slot types, and dialog-delegation configs from source (e.g., `interactionModel.json` for Alexa, `action.json` or `intents/*` for Google, App Intent declarations for Siri, YAML NLU configs for custom stacks). Treat as reference only — VOICE_IA is the target state, not a mirror of the current state.
5. **Decision documents** (optional) — architectural decisions that constrain the IA (e.g., "always confirm before any destructive action", "never give medical advice — always hand off", "account linking required for all personal-data intents").
6. **Sibling IAs** (optional) — if `MOBILE_IA.md`, `WEB_IA.md`, or `CLI_IA.md` exist, use-case traceability must be consistent across all channels. `MOBILE_IA.md` is especially important for App Intent overlap — App Intents / Siri Shortcuts invoked from *inside* a mobile app are owned by `MOBILE_IA.md`; full voice dialog architecture is owned here.
7. **Contract registry** (optional) — if present, per-intent data dependencies may reference endpoints by name; wire formats are owned by `CONTRACT_REGISTRY.md`, not redefined here.

The user may override scope (e.g., "only the Alexa skill, not the Google Action"). Support that by recording the narrowed scope in the Identity & Context section and excluding out-of-scope platforms from the inventory.

---

## Workflow

Voice IA generation proceeds in seven phases: scope, centralized-strategies (platform / invocation / slot / dialog), cross-cutting rules (confirmation / fallback / privacy), inventory, per-intent blueprint, flows and shared surfaces, traceability and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Scope & Audience

Derive from the product doc and use cases document the target voice surface (single platform, multi-platform, voice-only vs voice+screen, embedded in a mobile app vs standalone) and the audiences the IA serves (at-home, driving, hands-busy, accessibility-primary, multitasking). Record explicitly what is *not* in scope for this VOICE_IA document — text chatbots, App Intents owned by a sibling mobile IA, voice-writing copy, specific TTS voice selection. Keep audience treatment thin — named user types and their 3–7 top tasks per audience. No deep persona work.

### Phase 2: Centralized Strategies — Platform, Invocation, Slot, Dialog & Context

Decide the cross-cutting strategies before enumerating intents. This prevents per-intent drift where each intent invents its own slot types, confirmation policy, or context rules. Each strategy is declared exactly once (Rules 9–11) and referenced by name from every intent that uses it.

Decide, in order:

1. **Platform strategy** — which platforms are targeted; what feature-parity rules apply (identical across platforms vs platform-specific extensions); which platform-specific capabilities are in scope (Alexa Reminders API, Google App Actions, Siri App Intents, custom-stack features).
2. **Invocation model** — wake word / activation per platform; invocation name; bare-launch intent (opens the app with a greeting); deep-invocation phrasings ("ask {app} to {action}"); one-shot vs session-based interaction; session-end conditions (explicit stop, silence timeout, fulfilled intent with session-end, user-initiated exit).
3. **Slot model** — built-in slot types in use per platform (`AMAZON.Person`, `sys.date`, `sys.number`); custom slot definitions with canonical values and synonyms; open-ended-vs-enumerated rule per custom slot; slot-resolution confidence handling; entity-disambiguation strategy when multiple values match.
4. **Dialog model & context** — session scope (what is remembered within a single session and when it resets); cross-session scope (what is persisted per user with explicit privacy boundaries); context carryover for follow-up intents with explicit TTL; anaphora / deictic resolution ("that one", "the second one"); context-invalidation rules (new topic, explicit reset, timeout).

Use architecture-doc hints where available; fall back to conservative defaults. Surface non-obvious choices as open questions rather than guessing.

### Phase 3: Cross-Cutting Rules — Confirmation, Fallback & Handoff, Privacy & Safety

Declare, in order, three cross-cutting rule sets that every intent must respect:

1. **Confirmation strategy** — which intents (or classes of intents) always require explicit confirmation before fulfillment (destructive, irreversible, high-impact); which never require it (read-only, safe); confirmation-prompt role; confirmation-decline behavior.
2. **Fallback & handoff strategy** — fallback-intent behavior when no intent matches (generic don't-understand role, re-prompt limit, then exit); help-intent behavior (context-aware vs static); enumerated handoff targets (human operator, companion mobile app, web link, voicemail) and when each is used.
3. **Privacy & safety constraints** — what data is collected and retained (transient vs persisted, retention window); how sensitive topics are treated (medical / legal / financial / emergency — prefix / hand off / decline per topic); child-directed considerations if the app is kid-friendly or requires adult auth for specific intents; account-linking scope and revocation behavior.

Every per-intent blueprint downstream must respect these rules (Rules 12–14). A destructive intent without a confirmation, or an intent that violates the privacy constraints, is a rule violation.

### Phase 4: Intent Inventory

From use cases and architecture, enumerate the full set of **intent templates** (not sample utterances — see Rule 1). Classify each intent: action (mutates or fetches data), informational (answers a question), utility (`help`, `stop`, `cancel`, `repeat`, `launch`), or navigational (moves between flows or sections within a dialog). Build a single intent-inventory table that makes the hierarchy visible (group by area — core actions, informational, utility, navigational — via subsection headings or grouped rows).

For a new product, derive intents from use cases: every use case with a voice surface implies at least one intent where the user accomplishes or progresses that use case. For an existing codebase, discover current intents and reconcile against use cases — adding missing intents, flagging orphan intents.

### Phase 5: Per-Intent Blueprint

Iterate every intent template in the inventory and produce its full entry. This is the bulk of the output. Each entry must include every mandatory field listed in the per-intent blueprint template at [`references/per-intent-blueprint.md`](references/per-intent-blueprint.md) — read that file before writing the first entry; then copy the block per intent and fill in every field.

Work through intents in the order they appear in the inventory. When writing responses, describe each response by **role and content**, not by literal string (Rule 7). Draw every confirmation / failure-state behavior from the strategies declared in Phase 3. Reference slot types by name — do not re-enumerate (Rule 10).

### Phase 6: Dialog Flows & Shared Surfaces

Identify multi-turn dialog journeys that span multiple intents — onboarding, checkout, troubleshooting walkthroughs, account linking, multi-stage operations. For each, write a flow entry: ordered steps pointing to intents in the inventory, entry conditions, success exit, abort behavior, resume-on-return semantics, and the use case(s) the flow implements. Flows do not introduce new intents (Rule 15); they sequence existing ones and specify what state is passed between turns.

Identify shared surfaces that cross intents: re-prompt variants, fallback prompt, disambiguation sub-dialog, confirmation prompt, handoff-to-human prompt, account-linking prompt, session-closing prompts (graceful end, abrupt end on repeated failure). Consolidate these in the dedicated Shared Surfaces section (Rule 16). Per-intent entries may list which shared surfaces they can invoke by name, but must not re-describe them.

### Phase 7: Traceability & Validation

Build the bidirectional use-case ↔ intent matrix. Every use case in the input use cases document maps to at least one intent, or is explicitly listed as having no voice surface with a one-line reason. Every intent in the inventory is referenced by at least one use case, or is explicitly classified (utility, navigational, auth-linking). If sibling IAs are inputs, cross-reference use cases that surface on more than one channel.

Verify every rule holds: every action / informational intent has a primary response role, failure matrix complete per intent, every destructive intent has confirmation = yes (or explicit justification), every slot referenced in an intent exists in the Slot Model, every flow step references an existing intent, platform variations declared or "identical" stated, conditional sections either complete or absent. Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Intent templates, not sample utterances

An intent is one **template**; the sample utterances that invoke it are not separate intents. Document each intent once; list 5–15 representative sample utterances (with slot placeholders in braces) under that intent's blueprint; enumerate slot-dependent behavior in the blueprint's argument-dependent / failure-matrix fields. Do not fan out a separate entry per utterance phrasing.

### 2. Primary response role is mandatory per non-utility intent

Every action / informational / navigational intent must declare **exactly one** primary response role — the role the agent's successful response plays (fork-success confirmation, resource-status answer, flow-start acknowledgment, route-to-next-step transition). Utility intents (`help`, `stop`, `cancel`, `repeat`, `launch`) are classified as utility and carry only their inherent response role — no additional primary role is required. An intent that cannot name its primary response role is confused — either split it or re-examine its purpose.

### 3. Failure matrix is mandatory per intent

Every per-intent blueprint must enumerate, at minimum: fulfilled (primary response role delivered); slot unresolved after re-prompt limit (2–3 tries, then fallback); slot low-confidence or ambiguous (disambiguation sub-dialog triggered); permission required but not granted (account linking, location, personal-info consent — hand off to linking / consent flow); backend error (generic-failure response role); content filtered (query under a restricted topic — safety response role); out of scope (utterance matched a general template but requested action isn't supported — redirect-to-capability role); silence / no input (re-prompt strategy); session timeout; confirmation declined (for intents requiring confirmation). For each state describe **what the agent says (role)**, **what happens to the session (remain open / end)**, and **what is written to context**. "N/A — {reason}" is acceptable when a row genuinely does not apply (e.g., confirmation-declined on an intent that never confirms). Do not silently drop rows.

### 4. Every intent maps to at least one use case or is explicitly classified

An intent in the inventory either appears in the traceability matrix under at least one use case, or carries an explicit classification: utility, navigational, or auth-linking. Silent orphan intents are not allowed.

### 5. Every use case maps to at least one intent or is explicitly noted

Every use case in the input use cases document appears in the traceability matrix mapped to at least one intent, or is listed as "no voice surface" with a one-line reason (e.g., "web-only", "background job, no dialog"). Silent drops are not allowed.

### 6. No implementation detail

Do not write handler class structure, SDK-specific invocations, NLU model internals, ASR confidence-threshold code, or backend integration code. Those belong in per-unit SPECs. Describe intents by **invocation contract (utterances and slots), dialog behavior, response role, and failure states**, not by code shape.

**Worked example — the IA-vs-SPEC line.** This is the single most important boundary in the document; internalize it.

> **In scope (VOICE_IA):** "**Intent `ForkResource`**. Purpose: fork a resource to the user's namespace. Sample utterances: `fork {resource}`, `make a copy of {resource}`, `clone {resource} to my account`. Slot `resource` is required — type `ResourceName`; if missing, re-prompt with a slot-elicitation asking 'Which resource would you like to fork?'. Confirmation required before fulfillment — confirmation prompt role: 'confirm fork-target-and-destination summary'. Requires account linking; if not linked, hand off to linking-prompt shared surface. Primary response role: 'fork-success confirmation with follow-up suggestion'. Failure modes: slot unresolved after 2 re-prompts → fallback; resource not found → clarify-not-found response; backend error → generic-failure response with 'try again later' role. Context effect: remembers `last_fork` for 5 minutes for follow-up ('open it', 'delete it')."
>
> **Out of scope (VOICE_IA, belongs in SPEC):** "`ForkIntentHandler` implements `Handler` with `canHandle(HandlerInput)` checking `Alexa.getIntentName(request) === 'ForkResource'`; `handle(HandlerInput)` calls `ForkService.fork(slotValues.resource.resolutions.resolutionsPerAuthority[0].values[0].value.id)` and returns `responseBuilder.speak(…).withShouldEndSession(false).getResponse()`…"

Prefer contract-style descriptions over code-level descriptions throughout the output.

### 7. No final literal response copy

Describe response **roles** ("fork-success confirmation with follow-up suggestion", "not-found clarification", "re-prompt asking for the resource name") rather than exact spoken words ("Great — I forked it for you. Want to open it?"). One short placeholder phrase is acceptable for disambiguation only — no more. Final voice-writing is a downstream UX-writing phase.

### 8. No audio-aesthetic decisions

No specific TTS voice selection (Matthew vs Joanna), pitch settings, speaking-rate values, earcon sound choices, jingle or sting specifications. The existence and role of audio branding may be declared in the conditional Audio Branding section; specific audio content is owned downstream.

### 9. Invocation model stated once, then enforced

The Invocation Model section is authoritative. Every intent's launch and session behavior must conform to the declared model (wake word, invocation name, bare-launch vs deep-invocation, session-end rules). If an intent must deviate (e.g., an always-session-ending emergency intent), document the exception inline in the blueprint entry and explain why.

### 10. Slot model stated once, then referenced

The Slot Model section is authoritative. Per-intent blueprint entries reference slot types by name — "`resource` slot of type `ResourceName`" — and never re-enumerate canonical values, synonyms, or resolution rules. A per-intent entry may note slot-specific validation ("`resource` slot: accept only resources owned by the caller") without redefining the slot type itself.

### 11. Dialog model & context stated once, then referenced

The Dialog Model & Context section is authoritative. Per-intent blueprint entries document what they **read from context**, **write to context**, and **clear from context**, referencing context keys declared in the section. No per-intent entry redefines session-scope, cross-session-scope, TTL rules, or anaphora-resolution strategy.

### 12. Confirmation strategy stated once, enforced across intents

The Confirmation Strategy section is authoritative. Every per-intent blueprint declares `Confirmation required? yes` or `no`, and the value must be consistent with the strategy (every destructive or irreversible intent = yes; every safe / read-only intent = no, unless explicitly justified). A destructive intent with `Confirmation required? no` and no justification is a rule violation.

### 13. Fallback & handoff strategy stated once

The Fallback & Handoff Strategy section is authoritative. Per-intent blueprint entries reference fallback and handoff targets by name — "hand off to `human-operator` on repeated failure", "fall back to `generic-dontunderstand` after 2 re-prompts" — and never redefine the fallback or handoff flow itself.

### 14. Privacy & safety constraints stated once, respected by every intent

The Privacy & Safety Constraints section is authoritative. Every intent that collects, logs, or acts on user-personal data respects the declared retention and consent rules. Every intent whose utterances could trigger a sensitive topic (medical / legal / financial / emergency) declares its handling role in the failure matrix (content filtered → safety response role). Violations — an intent that logs data outside the retention window, or an intent that answers a medical question without the declared safety response — are rule violations.

### 15. Flows do not introduce new intents

Every flow step references an existing intent template in the inventory. If a flow needs a step that has no corresponding intent, add that intent to the inventory first (with a full blueprint), then reference it from the flow. Flows describe **sequencing and state-passing between intents**; they do not define new invocation contracts.

### 16. Shared surfaces are consolidated

Re-prompt variants, fallback prompt, disambiguation sub-dialog, confirmation prompt, handoff-to-human prompt, account-linking prompt, and session-closing prompts live in the Shared Surfaces section — not scattered across per-intent entries. Per-intent entries may list which shared surfaces they can invoke by name; they must not redefine them.

### 17. Conditional sections must be omitted, not left empty

If Multimodal Expansion, Account Linking, Audio Branding, Persistence & User Memory, Internationalization, Accessibility, or Telemetry & Analytics sections are not applicable for the project, omit them entirely. Do not leave them present with "N/A" content. Tight output is the house style.

### 18. Platform variations documented explicitly

When the app targets multiple platforms, every per-intent blueprint either declares `Platform variations: identical across platforms` or enumerates the divergence per platform (Alexa-only feature, Google-only behavior, Siri App Intent mapping). Silent platform divergence is not allowed. When only one platform is targeted, declare that in the Platform Strategy section and the per-intent field is redundant ("single-platform — no variations").

### 19. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact intent names, slot names, context keys, use-case references, and response roles. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: VOICE_IA.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
platforms: [{alexa, google, siri, custom} — subset]
intents_documented: {N}
utterances_total: {N}
slot_types_defined: {N}
flows_documented: {N}
shared_surfaces: {N}
auth_intents: {N}
confirmation_intents: {N}
use_cases_covered: {N}
use_cases_total: {N}
multimodal: {true | false}
open_questions: {N}
---

# VOICE_IA

> Single source of truth for the voice-driven conversational application's
> information architecture. Every intent template, sample utterance, slot,
> dialog flow, shared surface, and cross-cutting strategy is authoritative —
> per-unit voice SPECs must reference these entries.

## Identity & Context

{One paragraph: product name, the slice of the voice surface this document
 covers (platforms, voice-only vs voice+screen, standalone vs embedded), and
 the audience frame. Reference the product proposal by path. State explicitly
 what surfaces are out of scope — text chatbots, App Intents owned by
 MOBILE_IA, other channels, etc.}

## Audience & Top Tasks

| Audience | Usage context | Top tasks via voice |
|----------|---------------|---------------------|
| {audience name} | {at home / driving / hands-busy / accessibility-primary} | {3–7 comma-separated top tasks} |

(Repeat for each audience identified in the product doc.)

## Platform Strategy

- **Platforms targeted:** {list — e.g., Alexa, Google Assistant, Siri, custom}
- **Feature-parity rule:** {identical everywhere | per-platform divergence allowed | platform-specific features explicitly called out per intent}
- **Platform-specific capabilities in scope:** {Alexa-only features, Google-only features, Siri-only features, custom-stack features — or "None beyond common voice interaction."}
- **Primary platform:** {which platform drives the canonical behavior when platforms diverge, or "No primary — platforms treated as equal peers."}

## Invocation Model

- **Wake word / activation:** {per platform — "Alexa…", "Hey Google…", "Siri…", custom hotword, or push-to-talk}
- **Invocation name:** {the app's name as spoken — e.g., "open {app}", "ask {app} to…"}
- **Bare-launch intent:** {what happens on a bare launch — greeting role, help role, or "no bare launch; deep-invocation only"}
- **Deep-invocation phrasings:** {patterns — e.g., `ask {app} to {action}`, `tell {app} {slot}`}
- **One-shot vs session-based:** {rule — one-shot for simple intents, session-based for multi-turn}
- **Session-end conditions:** {explicit stop / cancel utterance, silence timeout with duration rule, fulfilled intent with session-end, user-initiated exit}

## Slot Model

### Built-in slot types in use

| Slot type | Platform | Purpose | Notes |
|-----------|----------|---------|-------|
| `{type}` | {platform} | {purpose} | {notes or "—"} |

### Custom slot types

| Type name | Canonical values (count) | Synonyms supported | Open-ended | Resolution strategy |
|-----------|--------------------------|--------------------|------------|---------------------|
| `{TypeName}` | {count + sample} | {yes/no} | {yes/no} | {rule} |

(Repeat for every custom slot type.)

### Low-confidence & disambiguation strategy

- **Low-confidence resolution:** {rule — e.g., "threshold below X → disambiguation sub-dialog; below Y → re-prompt; below Z → fallback"}
- **Multi-value disambiguation:** {how the dialog resolves multiple matching values — disambiguation sub-dialog with numbered list, or ask for qualifier}

## Dialog Model & Context

- **Session scope:** {what is remembered within a single session; when it resets}
- **Cross-session scope:** {what is persisted per user; privacy boundary; opt-in rule}
- **Context carryover keys:** list — each: name, source intent(s), TTL, invalidation rule
- **Anaphora / deictic resolution:** {rule — e.g., "'it', 'that one', 'the second' resolve against `last_resource`; fails gracefully with disambiguation if context is empty"}
- **Context invalidation rules:** {new topic, explicit reset ("start over"), timeout, session-end}

## Confirmation Strategy

- **Always-confirm classes:** {which intents or classes require explicit confirmation before fulfillment — e.g., destructive, irreversible, financial, >N-resource batch operations}
- **Never-confirm classes:** {read-only, idempotent safe reads}
- **Confirmation prompt role:** {describe what the prompt summarizes — target entity, action, consequences}
- **Confirmation-decline behavior:** {abort and return to prior state, offer alternative, end session}

## Fallback & Handoff Strategy

- **Fallback intent:** {behavior when no intent matches — generic-dontunderstand role; re-prompt limit; terminal exit behavior on repeated failure}
- **Help intent:** {static help response role, or context-aware help that adapts to current flow state}
- **Handoff targets:** list — each: name (human-operator / companion-mobile-app / web-link / voicemail), trigger condition, transition role, state carried over

## Privacy & Safety Constraints

- **Data collected & retention:** {transient-only vs persisted; retention window for persisted; logging rule}
- **Sensitive topics:** list — each topic (medical / legal / financial / emergency / other): handling rule (prefix with disclaimer / always hand off / always decline with role)
- **Child-directed considerations:** {if applicable — adult-auth gate for specific intent classes, simplified language policy}
- **Account-linking scope:** {what account-linking grants the app access to; revocation behavior; re-link triggers}

## Intent Inventory

| Intent | Kind | Auth | Confirmation | Multimodal | Use cases | Purpose |
|--------|------|------|--------------|------------|-----------|---------|
| `{IntentName}` | action / informational / utility / navigational | none / account-linked / permission-granted | yes / no | voice-only / voice+screen / companion-push | {UC-IDs or classification} | {one line} |

(Group by area — core actions, informational, utility, navigational — via
 subsection headings or visible row grouping. Repeat for every intent template.)

---

## Per-Intent Blueprints

Every intent template in the inventory gets a full blueprint entry here. The
blueprint template — with every required field, its purpose, and common
mistakes to avoid — lives in
[`references/per-intent-blueprint.md`](references/per-intent-blueprint.md).
Read that file before writing the first entry; then copy the block per intent
and fill in every field.

Entries in the output appear in the order they appear in the Intent Inventory,
grouped by the same area headings.

### `{IntentName}` — {short purpose}

{Full blueprint block per `references/per-intent-blueprint.md`.}

---

(Repeat the blueprint block for every intent template in the inventory.)

---

## Dialog Flows

### {Flow name}

- **Purpose:** {one sentence}
- **Use cases implemented:** {UC-IDs}
- **Entry condition:** {what triggers the flow — e.g., "user invokes `StartCheckout` in a linked session"}
- **Steps:**
  1. `{IntentName}` — {what the user says here; what state is produced for the next step}
  2. `{IntentName}` — {what the user says here}
- **Success exit:** {terminal state — primary response role delivered, session-end behavior, context written}
- **Abort behavior:** {explicit `stop` / `cancel` mid-flow; what is preserved; where the user lands}
- **Resume semantics:** {re-invoke intent? resume token? start from last completed step? timeout before state is dropped}

(Repeat for each flow. If none: "No multi-turn flows in scope.")

---

## Shared Surfaces

| Surface | Kind | Trigger | Dismissal / completion | Invoked by intents |
|---------|------|---------|------------------------|--------------------|
| {name} | re-prompt / fallback / disambiguation / confirmation / handoff / account-linking / session-closing | {what opens it} | {how it closes} | {intent names or "multiple"} |

(Repeat for every shared surface. If none: "No shared surfaces in scope.")

---

## Multimodal Expansion

(Omit this section entirely if the app targets voice-only devices.)

- **Target device classes:** {Echo Show, Nest Hub, car display, phone-with-screen — list}
- **Visual card conventions:** {when a card appears; card content roles (title, body, image, structured list); when no card}
- **Touchable control conventions:** {when on-screen buttons / selectors appear alongside voice; fallback when touch is unavailable}
- **Media playback conventions:** {audio / video playback rules; background-audio behavior}
- **Companion-push conventions:** {when the dialog hands off to the companion mobile app via push notification; what payload is passed}

Per-intent blueprint entries declare their multimodal expansion (visual card
role, touch controls, companion-push behavior, or "voice-only").

---

## Account Linking

(Omit this section entirely if no intents require account linking.)

- **Linking flow model:** {OAuth device flow, OAuth authorization code with redirect, custom skill-account-linking}
- **Landing page pattern:** {URL convention for the companion linking page, reference to WEB_IA if present}
- **Token exchange & scopes:** {scopes requested; refresh-token policy}
- **Unlink / revocation behavior:** {what happens on unlink; what intents fail after unlink; re-link prompt role}

---

## Audio Branding

(Omit this section entirely if audio branding is out of scope.)

- **Earcons in use:** {existence and role only — e.g., "launch-earcon on bare launch", "success-earcon on destructive-intent fulfillment". Specific sounds owned downstream.}
- **Music / stings:** {existence and role only.}
- **When audio branding is suppressed:** {accessibility mode, voice-only scripting context, etc.}

---

## Persistence & User Memory

(Omit this section entirely if no cross-session persistence.)

- **Memory kinds:** {preferences, last-used params, history, favorites — list}
- **Scope & TTL:** {per-kind retention window and invalidation rule}
- **User-initiated reset:** {utterance or setting that clears memory; which intents honor it}
- **Privacy boundary:** {what is never persisted}

---

## Internationalization

(Omit this section entirely if single-locale.)

- **Locales supported:** {list}
- **Utterance translation strategy:** {human-authored per locale; shared template with per-locale slot synonyms; machine translation with review}
- **Locale-specific slot types:** {date formats, number formats, name-entity sets per locale}
- **Invocation-name localization:** {per-locale invocation name, or shared name}

---

## Accessibility

(Omit this section entirely if there are no accommodations beyond default voice interaction.)

- **Slower speech rate on request:** {utterance or setting}
- **Screen-reader co-presence on multimodal devices:** {rule when both voice and assistive tech are active}
- **Cognitive accessibility:** {simplified-language mode, longer silence timeouts, repeat-prompt behavior}

---

## Telemetry & Analytics

(Omit this section entirely if no telemetry is in scope for this document.)

- **Event naming:** {convention — e.g., `intent_invoked`, `intent_fulfilled`, `intent_fallback`; property schema at a high level, not full keys}
- **Funnel tracking:** {which flows are funnel-tracked; at which steps events fire}
- **Privacy contract:** {what is never sent — slot values, user utterances in raw form, etc.}

---

## Traceability Matrix

### Use cases → intents

| Use case | Intent(s) | Notes |
|----------|-----------|-------|
| {UC-ID} | `{IntentName}`, `{IntentName}` | {brief note, or "Primary surface."} |
| {UC-ID} | — | No voice surface — {one-line reason}. |

(Every use case from the input document appears here. When sibling IAs exist,
 note in the third column which other channels also surface this use case.)

### Intents → use cases

| Intent | Use case(s) | Classification if none |
|--------|-------------|------------------------|
| `{IntentName}` | {UC-IDs} | — |
| `{IntentName}` | — | utility / navigational / auth-linking |

(Every intent from the inventory appears here.)

---

## Relationship to Other Documents

- **Product proposal and use cases:** This document realizes use cases on the voice surface. Use-case IDs in the traceability matrix match the source use cases document.
- **Per-unit SPECs:** SPECs for voice units must reference the VOICE_IA entry for the intent(s), flow(s), or shared surface(s) they implement, and use intent names, slot names, context keys, and response roles from this document verbatim.
- **CONTRACT_REGISTRY.md** (if present): per-intent backend data dependencies may reference endpoint names; wire-format shapes are owned by the contract registry, not redefined here.
- **MOBILE_IA.md** (if present): App Intents / Siri Shortcuts invoked from inside the mobile app are owned there; full voice dialogs invoked via Alexa / Google / Siri standalone are owned here. The traceability matrix cross-references use cases that span both.
- **WEB_IA.md** (if present): the account-linking landing page and any voice-app marketing pages are owned there; voice-surface IA is owned here.
- **CLI_IA.md** (if present): parallel IA for terminal surfaces; traceability must be consistent.
- **Voice-writing / UX-writing phase:** final spoken copy (exact words, tone, humor, personality) is owned downstream.
- **DESIGN-equivalent audio execution:** specific TTS voice, speaking rate, earcon sounds, and musical stings are owned downstream.

---

## Open Questions

- [ ] {Question — e.g., "Should the `ForkResource` intent default to confirming the destination namespace, or assume the caller's personal namespace and only confirm on non-default targets?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Full intent-template inventory with classification (action / informational / utility / navigational), sample utterances, and slot references
- Per-intent blueprint: purpose, audience, auth, sample utterances, slots, confirmation requirement, primary response role, alternative response variations, multimodal expansion, data dependencies, context read / written / cleared, full failure matrix, handoff paths, platform variations, session-end behavior
- Invocation model, slot model, dialog model & context, confirmation strategy, fallback & handoff strategy, privacy & safety constraints — each declared exactly once as cross-cutting strategy
- Multi-turn dialog flows that sequence intents from the inventory
- Shared surface inventory (re-prompt, fallback, disambiguation, confirmation, handoff, account-linking, session-closing)
- Platform strategy when multiple platforms are targeted
- Bidirectional traceability between use cases and intents
- Conditional sections (multimodal expansion, account linking, audio branding, persistence & user memory, i18n, accessibility, telemetry & analytics) when applicable to the product

### Out of scope

- Implementation details — handler class structure, SDK-specific invocations, NLU model internals, ASR confidence thresholds in code, backend integration code — owned by per-unit `SPEC.md`
- Final literal spoken copy (exact words, prompts, error messages, personality, humor) — owned by a voice-writing / UX-writing phase
- Audio aesthetics — specific TTS voice selection, speaking rate, earcon sounds, jingles, stings — owned downstream
- Backend / API wire formats — owned by `CONTRACT_REGISTRY.md`
- Web information architecture — owned by `WEB_IA.md`
- Mobile-app information architecture — owned by `MOBILE_IA.md`, including App Intents / Siri Shortcuts invoked from inside the mobile app
- CLI information architecture — owned by `CLI_IA.md`
- Text chatbot surfaces — owned by a future `CHAT_IA.md` (unless explicitly scoped in via a shared intent model, declared in Phase 1)
- ASR / NLU / TTS model training, dataset curation — owned by research / ML docs
- Database schema, business logic, algorithm design — owned by architecture and unit SPECs
- Device onboarding, smart-home pairing, hardware setup — owned by DEPLOYMENT or device-setup docs
- Persona research and user research — owned by a separate research phase

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `platforms`, `intents_documented`, `utterances_total`, `slot_types_defined`, `flows_documented`, `shared_surfaces`, `auth_intents`, `confirmation_intents`, `use_cases_covered`, `use_cases_total`, `multimodal`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Every intent in the Intent Inventory table has a full per-intent blueprint entry below
- [ ] Every per-intent entry declares kind, auth requirement, 5–15 sample utterances with slot placeholders, slots (each referencing the Slot Model), confirmation requirement, primary response role (or utility classification), alternative response variations, multimodal expansion (or "voice-only"), data dependencies, context read / written / cleared, full failure matrix, handoff paths, platform variations (or "identical"), session-end behavior
- [ ] Invocation Model, Slot Model, Dialog Model & Context, Confirmation Strategy, Fallback & Handoff Strategy, and Privacy & Safety Constraints each appear in exactly one section; per-intent entries reference them and do not redefine them
- [ ] Every destructive or irreversible intent has `Confirmation required: yes` (or explicit justification inline in the blueprint)
- [ ] Every intent respects the Privacy & Safety Constraints — no intent logs outside the retention window, no intent answers a sensitive topic without the declared safety role
- [ ] Every slot referenced in an intent exists in the Slot Model (built-in or custom)
- [ ] Every context key written or read by an intent exists in the Dialog Model & Context section's carryover key list
- [ ] Every use case in the input use cases document maps to at least one intent in the traceability matrix, or is explicitly listed as having no voice surface with a one-line reason
- [ ] Every intent in the inventory is referenced by at least one use case in the traceability matrix, or is explicitly classified (utility, navigational, auth-linking)
- [ ] Every flow step references an existing intent template from the inventory (no flow introduces new intents)
- [ ] Shared surfaces appear only in the dedicated Shared Surfaces section, not scattered in per-intent entries
- [ ] Platform variations are declared explicitly in every per-intent blueprint when multiple platforms are targeted; "identical across platforms" stated when no divergence
- [ ] Conditional sections (multimodal expansion, account linking, audio branding, persistence & user memory, i18n, accessibility, telemetry & analytics) are either fully completed or fully omitted — never present-and-empty
- [ ] No implementation detail (handler classes, SDK calls, NLU internals, ASR code) appears anywhere in the document
- [ ] No audio-aesthetic decisions (specific TTS voice, pitch, speaking rate, earcon sounds, jingle content) appear anywhere
- [ ] No final literal response copy — response roles are described, not exact spoken strings; at most one short placeholder phrase per entry for disambiguation
- [ ] Frontmatter `use_cases_covered` and `use_cases_total` are present; any gap is explained in the Traceability Matrix
- [ ] If sibling IAs (`MOBILE_IA.md`, `WEB_IA.md`, `CLI_IA.md`) exist, the traceability matrix cross-references use cases spanning multiple channels
