---
name: MOBILE_IA.md
description: Generate a native mobile information architecture registry — platform strategy, navigation model, deep-link strategy, permissions strategy, screen-template inventory, per-screen blueprints, flows, shared surfaces, push notifications, lifecycle behavior, and use-case traceability. Use when asked to create a mobile IA, design the app navigation and screen map, produce a screen inventory, define deep-link and permissions strategy, or produce a MOBILE_IA.md.
---

# Task: Generate MOBILE_IA.md — Native Mobile Information Architecture Registry

## Objective

Produce a MOBILE_IA.md that serves as the single source of truth for a native mobile application's information architecture: the platform strategy, navigation model (stack / tab / drawer / modal), deep-link strategy, permissions strategy, full screen-template inventory, per-screen blueprint (purpose, sections, primary CTA, data dependencies, gestures, orientation, permissions, platform variations, state matrix including offline and permission-denied, entry/exit points, lifecycle behavior, parameter-dependent behavior), cross-screen flows, shared native surfaces, push-notification architecture, terminal and background lifecycle behavior, global states, orientation and form-factor strategy, accessibility commitments, and bidirectional traceability from product use cases to screens. An agent or designer reading this document knows exactly what screens exist, how users move between them, what permissions and platform behavior each screen depends on, and how every use case is realized on the native mobile surface — without making any independent architectural decisions. MOBILE_IA sits between product requirements (proposal, use cases) and per-unit mobile SPECs, the same way WEB_IA and CLI_IA do for their respective surfaces.

---

## Inputs

1. **Product / proposal document** (required) — product purpose, target users, and rationale. Sets the scope and audience frame for the IA.
2. **Use cases document** (required) — enumerates user interactions and jobs-to-be-done. Primary input. Every use case with a native-mobile surface must map to one or more screens in the inventory.
3. **Architecture document** (optional but recommended) — platform choice (native iOS Swift/SwiftUI, native Android Kotlin/Compose, React Native, Flutter, Kotlin Multiplatform), navigation library, state-management choice, offline strategy, auth model, minimum OS versions. Informs platform strategy and navigation conventions. If absent, infer conservative defaults and flag non-obvious choices as open questions.
4. **Existing mobile codebase** (auto-discovered) — discover current screens, navigation graph, deep-link schemes, and declared permissions from source (e.g., `NavGraph.kt`, `AppCoordinator.swift`, route tables, `AndroidManifest.xml`, `Info.plist`). Treat as reference only — MOBILE_IA is the target state, not a mirror of the current state.
5. **Decision documents** (optional) — architectural decisions that constrain the IA (e.g., "biometric re-auth required after 5 minutes background", "all destructive actions require native confirmation alert").
6. **Sibling IA documents** (optional) — `WEB_IA.md`, `CLI_IA.md`, `VOICE_IA.md`, `TUI_IA.md`. If present, use-case traceability must be consistent across channels. Every use case surfaces on at least one channel or is explicitly classified as having none.
7. **Contract registry** (optional) — if present, per-screen data dependencies may reference endpoints by name; wire formats are owned by CONTRACT_REGISTRY.md, not redefined here.

The user may override scope (e.g., "iOS only, defer Android", or "phone only, defer tablet"). Support that by recording the narrowed scope in the Identity & Context section and excluding out-of-scope surfaces from the inventory.

Scope note: this skill covers **native mobile surfaces** — apps distributed through App Store or Google Play, running on iOS or Android, built natively or with cross-platform frameworks (React Native, Flutter, Kotlin Multiplatform). WebView-wrapped web apps are out of scope — use `WEB_IA.md` for those. Terminal-invoked binaries are in `CLI_IA.md`. Full voice UIs are in `VOICE_IA.md` (but App Intents / Siri Shortcuts / Google Assistant actions as mobile-app invocation points are in scope here).

---

## Workflow

Mobile IA generation proceeds in seven phases: scope, platform-and-navigation globals, inventory, per-screen blueprint, flows and shared surfaces, push and lifecycle and global states, traceability and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Scope & Audience

Derive from the product doc and use cases document the target mobile surface (iOS only, Android only, both; phone only, phone and tablet; with or without wearable / CarPlay / widget companions) and the audiences the IA serves. Record explicitly what is *not* in scope for this MOBILE_IA document — WebView wrappers, non-mobile surfaces, companion platforms the team is deferring. Keep audience treatment thin — named user types and their 3–7 top tasks per audience. No deep persona work.

### Phase 2: Platform Strategy, Navigation Model, Deep-Link Strategy, Permissions Strategy

Lock the four centralized sections before enumerating screens. This prevents per-screen drift where each screen reinvents its navigation placement, deep-link shape, or permission handling.

**Platform Strategy.** Decide:
- Platforms covered (iOS / Android / both)
- Native vs cross-platform framework
- Minimum OS versions per platform
- Feature-parity rules — what must behave identically across platforms, where platform-idiomatic divergence is allowed

**Navigation Model.** Decide, as a single authoritative description:
- Root container — tab bar, navigation drawer, single stack
- Per-tab navigation stacks — each tab's independent push/pop history (if tabbed)
- Modal presentation conventions — full-screen modal, iOS sheet, bottom sheet, alert / dialog
- Back semantics — iOS swipe-back + nav-bar back, Android hardware/gesture back, what back does per screen kind (pop, dismiss modal, close app)
- State restoration — what survives app kill, what survives backgrounding

**Deep-Link Strategy.** Decide:
- URL scheme (e.g., `tool://...`)
- Universal Links (iOS) / App Links (Android) — associated domains
- Auth-gated deep-link behavior — route to login with preserved intended destination
- Destination coverage — which screens are reachable via deep link

**Permissions Strategy.** Decide:
- Which permissions the app requests (camera, microphone, location precise/coarse, photos, contacts, notifications, Bluetooth, Health, etc.)
- When each is prompted — cold start, first-use of feature, explicit settings flow
- Rationale copy strategy — pre-prompt explainer screen vs direct system prompt
- Denial behavior per permission — feature degraded gracefully, feature blocked with Settings CTA, app-level blocker
- Re-request rules — deep-link-to-Settings escape hatch

Use architecture-doc hints where available; fall back to conservative defaults. Surface non-obvious choices as open questions rather than guessing.

### Phase 3: Screen Inventory

From use cases and architecture, enumerate the full set of **screen templates** (not screen instances — see Rule 1). Classify each screen: authenticated vs public, audience served, chrome / nav-container / auth-gate / content / error / splash / onboarding-slide category. Build a single inventory table with navigation position (which tab, which stack, modal vs pushed), platform availability, auth requirement, audience, use cases, and a one-line purpose.

For a new product, derive screens from use cases: every use case with a mobile surface implies at least one screen where the user completes or progresses that use case. For an existing codebase, discover current screens from the navigation graph and reconcile against use cases — adding missing screens, flagging orphan screens.

### Phase 4: Per-Screen Blueprint

Iterate every screen template in the inventory and produce its full entry. This is the bulk of the output. Each entry must include every mandatory field listed in the per-screen blueprint template. The blueprint template — with every required field, its purpose, and common mistakes to avoid — lives in [`references/per-screen-blueprint.md`](references/per-screen-blueprint.md). Read that file before writing the first entry; then copy the block per screen and fill in every field.

Work through screens in the order they appear in the inventory, grouped by area. When writing sections/blocks, describe each region by **role and content**, not by component name or visual appearance (see Rule 6).

### Phase 5: Flows & Shared Surfaces

Identify multi-screen user journeys — onboarding, signup with biometric enrollment, checkout with native pay, permission dances, password reset, multi-step forms. For each, write a flow entry: ordered steps pointing to screens in the inventory, entry conditions, success exit, abort behavior, resume-on-background semantics, and the use case(s) the flow implements. Flows do not introduce new screens (Rule 12); they sequence existing ones.

Identify shared native surfaces that cross screens: alerts / dialogs, action sheets / bottom sheets, toasts / snackbars, share sheets, system pickers (photo, document, contact), in-app review prompts, biometric prompts, pull-to-refresh, skeleton loaders, empty-state patterns. Consolidate these in the dedicated Shared Surfaces section (Rule 13). Per-screen entries may list which shared surfaces they invoke by name; they must not re-describe them.

### Phase 6: Push Notifications, Lifecycle, Global States, Orientation & Form Factor, Accessibility

Document the remaining cross-screen systems.

**Push Notifications.** Categorize notifications (transactional, reminder, promotional, safety). For each category: trigger, routing destination (which screen, with what params), rich media, action buttons, platform differences (iOS Notification Extensions, Android channels), silent / data-only payloads and what they update. Document the permission-request strategy (when asked, pre-prompt rationale).

**Lifecycle Behavior.** State the default cold-start / warm-start / backgrounded / killed behavior globally; per-screen deltas go in blueprint entries. Cover: timers paused on background, uploads continued via background tasks, biometric re-auth on resume, state-restoration strategy (none / deep-link-only / full restoration).

**Global States.** Offline banner, update-required (force upgrade), maintenance mode, feature-flag gating, app-version deprecation. One pattern per row — what surface the user sees, what actions are available.

**Orientation & Form Factor.** Portrait / landscape rules at the app level; phone vs tablet strategy (iPad split-view, foldables); what is optimized vs what just stretches.

**Accessibility.** VoiceOver / TalkBack semantic labels, focus order, dynamic type support, switch control, accessibility shortcuts. Mobile accessibility is mandatory in a way it is not on web — treat this section as required unless explicitly descoped.

### Phase 7: Traceability & Validation

Build the bidirectional use-case ↔ screen matrix. Every use case in the input use cases document maps to at least one screen, or is explicitly listed as having no mobile surface with a one-line reason. Every screen in the inventory is referenced by at least one use case, or is explicitly classified (chrome, nav-container, auth-gate, error, splash, onboarding-slide, informational). If sibling IAs exist (`WEB_IA.md`, `CLI_IA.md`, `VOICE_IA.md`, `TUI_IA.md`), cross-reference use cases that surface on multiple channels.

Verify every rule holds: navigation model stated once, deep-link strategy stated once, permissions strategy stated once, full state matrix per screen, primary CTA declared (or explicit classification), platform variations documented (or "identical" stated), no implementation detail, no visual-design decisions, no final copy, conditional sections either complete or absent. Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Screen templates, not screen instances

A parameterized screen like `ResourceDetail(resourceId)` is one screen template, not N screens. Document each template once. Note the parameters and enumerate parameter-dependent behavior (e.g., "private resource + viewer has no access → not-found screen with 'Request access' CTA, not blank error"). Do not enumerate concrete instances.

### 2. Primary CTA is mandatory per non-chrome screen

Every screen must declare **exactly one** primary CTA, or be explicitly classified as a chrome / nav-container / auth-gate / error / splash / onboarding-slide / informational screen with no CTA. A screen that cannot name its primary CTA is confused — either split it or re-examine its purpose.

### 3. State matrix is mandatory per screen

Every per-screen blueprint must enumerate, at minimum: default / loaded-with-data; loading (cold start, warm start, subsequent refresh); empty; error (distinguish transient network from hard error); offline (mobile-specific — cached content? disabled actions? banner?); unauthenticated (if auth-gated); unauthorized; not-found (for parameterized screens); permission-denied (mobile-specific — for every permission the screen depends on, what is shown and what CTA is available); backgrounded (state when the app is backgrounded mid-screen — preserved? reset? timer cancelled?). For each state describe **what the user sees and what actions are available**. Do not describe colors, typography, or animation curves.

### 4. Every screen maps to at least one use case or is explicitly classified

A screen in the inventory either appears in the traceability matrix under at least one use case, or carries an explicit classification: chrome, nav-container, auth-gate, error, splash, onboarding-slide, informational. Silent orphan screens are not allowed.

### 5. Every use case maps to at least one screen or is explicitly noted

Every use case in the input use cases document appears in the traceability matrix mapped to at least one screen, or is listed as "no mobile surface" with a one-line reason (e.g., "web-only", "background job, no UI", "CLI-only tooling"). Silent drops are not allowed.

### 6. No implementation detail

Do not write view-model class names, state-flow types, dependency-injection wiring, navigation-library route-builder APIs, or function signatures. Those belong in per-unit SPECs. Describe screens by **section roles and behavior contracts**, not by code shape.

**Worked example — the IA-vs-SPEC line.** This is the single most important boundary in the document; internalize it.

> **In scope (MOBILE_IA):** "The Resource Detail screen shows the resource header (name, owner avatar, primary 'Fork' CTA), a tabbed section switcher (Overview / Files / Activity), and a context action sheet triggered by the overflow button. On iOS the action sheet is a bottom sheet; on Android it is a Material bottom sheet. Pull-to-refresh is supported. The screen supports portrait only. Requires authenticated user. Deep-linkable via `tool://resource/{id}`."
>
> **Out of scope (MOBILE_IA, belongs in SPEC):** "`ResourceDetailViewModel` exposes `state: StateFlow<ResourceDetailState>` and `actions: SharedFlow<ResourceDetailAction>`; the `Fork` intent maps to `ForkUseCase.execute(resourceId)` and dispatches `ForkResult.Success | ForkResult.Conflict`…"

Prefer section-and-role descriptions over class-and-field descriptions throughout the output.

### 7. No visual-design decisions

No specific colors, font families, exact spacing, iconography, illustration selections, animation curves, or platform-library theme tokens. HIG / Material compliance is an execution-time concern, not an IA concern. Platform-idiomatic **behavior** (iOS sheet vs Android bottom sheet, swipe-back vs system back) is in scope; platform-idiomatic **aesthetics** (specific system colors, typography tokens, animation timings) are not.

### 8. No final copy

Describe the role of text ("primary CTA for forking the resource", "auth-failure message", "pre-prompt rationale for camera access") rather than literal strings ("'Fork Repository'"). Short disambiguating placeholders (e.g., `[Fork]`) are acceptable — no more.

### 9. Navigation model is stated once, then enforced

The Navigation Model section is authoritative. Every screen in the inventory must conform to the declared model — its navigation position (which tab, which stack, modal vs pushed, alert vs sheet) must reference the model, not invent new containers. If a screen suppresses a global chrome element (e.g., a focused onboarding step hides the tab bar), note the suppression in the entry.

### 10. Deep-link strategy is stated once, then enforced

The Deep-Link Strategy section is authoritative. Every deep-linkable screen's URL pattern must conform to the declared scheme and universal-link conventions. Per-screen entries may declare "not deep-linkable" — silent omission is not allowed. If auth-gated, the strategy's auth-gate behavior applies; per-screen entries reference it rather than re-describing it.

### 11. Permissions strategy is stated once, then enforced

The Permissions Strategy section is authoritative. Per-screen entries reference permissions **by name** from the centralized list (e.g., "camera", "location-precise"). They do not re-describe the prompting strategy, rationale copy, or denial handling. Every permission referenced in a per-screen entry must appear in the Permissions Strategy section.

### 12. Flows do not introduce new screens

Every flow step references an existing screen template in the inventory. If a flow needs a step that has no corresponding screen, add that screen to the inventory first (with a full blueprint), then reference it from the flow.

### 13. Shared surfaces are consolidated

Alerts, dialogs, action sheets, bottom sheets, toasts, snackbars, share sheets, system pickers, in-app review prompts, biometric prompts, pull-to-refresh, skeleton loaders, and empty-state patterns live in the Shared Surfaces section — not scattered across per-screen entries. Per-screen entries may list which shared surfaces they can invoke by name; they must not redefine them.

### 14. Conditional sections must be omitted, not left empty

If App Store Presence, Home-Screen Widgets & Lock-Screen, App Intents / Siri Shortcuts / Google Assistant, Wearable Companions, CarPlay / Android Auto, Internationalization, or Telemetry sections are not applicable for the project, omit them entirely. Do not leave them present with "N/A" content. Tight output is the house style.

### 15. Platform variations are documented explicitly or declared identical

Every per-screen blueprint must either enumerate the per-platform divergences (iOS vs Android) or explicitly state "identical across platforms". Silent equivalence is not allowed — the reader must know whether a screen behaves the same on both platforms or diverges, and how.

### 16. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact screen names, route patterns, permission names, exit-code categories, and use-case references. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: MOBILE_IA.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
platforms: {[ios, android] | [ios] | [android] | [cross-platform]}
screens_documented: {N}
flows_documented: {N}
shared_surfaces: {N}
deep_linkable_screens: {N}
permissions_requested: {N}
push_categories: {N}
use_cases_covered: {N}
use_cases_total: {N}
auth_screens: {N}
open_questions: {N}
---

# MOBILE_IA

> Single source of truth for the native mobile application's information architecture.
> Every screen template, navigation rule, deep link, permission, flow, shared surface,
> and push-notification category is authoritative — per-unit mobile SPECs must reference
> these entries.

## Identity & Context

{One paragraph: product name, the slice of the native-mobile surface this document
 covers (iOS, Android, or both; phone only or phone+tablet; native or cross-platform),
 and the audience frame. Reference the product proposal by path. State explicitly what
 surfaces are out of scope — WebView-wrapped experiences, wearable/CarPlay if deferred,
 non-mobile channels covered by sibling IA documents.}

## Audience & Top Tasks

| Audience | Top tasks on mobile |
|----------|---------------------|
| {audience name} | {3–7 comma-separated top tasks} |

(Repeat for each audience identified in the product doc.)

## Platform Strategy

- **Platforms covered:** {iOS | Android | both}
- **Framework:** {native iOS Swift/SwiftUI | native Android Kotlin/Compose | React Native | Flutter | Kotlin Multiplatform — with rationale}
- **Minimum OS versions:** {iOS {version}; Android API {level}}
- **Feature-parity rule:** {what must behave identically across platforms}
- **Platform-idiomatic divergence rule:** {where platform-native behavior is allowed to differ — e.g., iOS sheet vs Android bottom sheet, iOS swipe-back vs Android system back}

## Navigation Model

- **Root container:** {tab bar | navigation drawer | single stack — with rationale}
- **Per-tab stacks:** {list of tabs, each with its independent stack scope — or "N/A — no tabs"}
- **Modal presentation:** {conventions — full-screen modal, iOS sheet, bottom sheet, alert / dialog; which screen kinds use which}
- **Back semantics:**
  - **iOS:** {swipe-back behavior, nav-bar back button behavior}
  - **Android:** {hardware / gesture back behavior; what back does at stack root — close tab, close app, confirm exit}
- **State restoration:**
  - **Backgrounded:** {what is preserved — navigation stack, scroll position, in-progress input, timers}
  - **Killed:** {strategy — none | deep-link-only | full restoration}

## Deep-Link Strategy

- **URL scheme:** `{scheme}://{...}` — {when used}
- **Universal Links / App Links:** {associated domains — or "None."}
- **Auth-gated deep-link behavior:** {rule — e.g., "If unauthenticated, route to login; preserve intended destination; resume after auth."}
- **Destination coverage:** {which screens are reachable via deep link — enumerated in the Screen Inventory's deep-link column}
- **Exceptions:** {any screens that deviate, with reason — or "None."}

## Permissions Strategy

| Permission | When prompted | Rationale strategy | Denial behavior |
|------------|---------------|--------------------|-----------------|
| `{name}` | {cold start / first-use / explicit settings flow} | {pre-prompt explainer | direct system prompt} | {graceful degrade | feature-blocked with Settings CTA | app-level blocker} |

(Repeat for every permission the app requests. If none: "No permissions requested beyond the platform defaults.")

- **Re-request rule:** {system re-prompt policy; Settings deep-link as escape hatch}

## Screen Inventory

| Screen | Nav position | Platforms | Auth | Deep link | Audience | Use cases | Purpose |
|--------|--------------|-----------|------|-----------|----------|-----------|---------|
| `{ScreenName}` | {tab / stack / modal / alert} | {iOS / Android / both} | {public | auth | auth-with-permission} | {URL pattern or "—"} | {audience} | {UC-IDs or classification} | {one line} |

(Group by area — onboarding, authenticated app, settings, etc. — via subsection
 headings or visible indentation. Repeat for every screen template.)

---

## Per-Screen Blueprints

Every screen template in the inventory gets a full blueprint entry here. The
blueprint template — with every required field, its purpose, and common
mistakes to avoid — lives in
[`references/per-screen-blueprint.md`](references/per-screen-blueprint.md).
Read that file before writing the first entry; then copy the block per screen
and fill in every field.

Entries in the output appear in the order they appear in the Screen Inventory,
grouped by the same area headings.

### `{ScreenName}` — {short name}

{Full blueprint block per `references/per-screen-blueprint.md`.}

---

(Repeat the blueprint block for every screen template in the inventory.)

---

## Flows

### {Flow name}

- **Purpose:** {one sentence}
- **Use cases implemented:** {UC-IDs}
- **Entry condition:** {what triggers the flow — e.g., "first cold start with no stored auth"}
- **Steps:**
  1. `{ScreenName}` — {what the user does here; what state is produced for the next step}
  2. `{ScreenName}` — {what the user does here}
- **Success exit:** {terminal state — e.g., "user lands on Home tab, authenticated, notification permission resolved"}
- **Abort behavior:** {can the user leave? what is preserved? where do they land?}
- **Resume on background:** {what happens if the user backgrounds the app mid-flow — resume where left off, restart, timeout?}

(Repeat for each flow. If none: "No multi-screen flows in scope.")

---

## Shared Surfaces

| Surface | Kind | Trigger | Dismissal | Platform variations | Invoked by screens |
|---------|------|---------|-----------|---------------------|---------------------|
| {name} | {alert / action sheet / bottom sheet / toast / snackbar / share sheet / photo picker / document picker / contact picker / in-app review / biometric prompt / pull-to-refresh / skeleton loader / empty-state} | {what opens it} | {how it closes} | {iOS vs Android differences, or "identical"} | {screen names or "global"} |

(Repeat for every shared surface. If none: "No shared surfaces in scope.")

---

## Push Notifications

(If the app sends no notifications, replace this section with: "No push notifications in scope." and state so in Identity & Context.)

- **Permission request strategy:** {when asked — at cold start, deferred to first-use, never; pre-prompt rationale screen or direct system prompt}

### Categories

| Category | Trigger | Routing destination | Rich media | Action buttons | Platform differences |
|----------|---------|---------------------|------------|----------------|----------------------|
| {transactional / reminder / promotional / safety} | {what causes it to be sent} | `{ScreenName}` with {params} | {image / video / none} | {inline actions, or "None"} | {iOS-specific behavior, Android channel, or "identical"} |

(Repeat for every notification category.)

- **Silent / data-only notifications:** {what the app updates when it receives a silent payload, or "None."}

---

## Lifecycle Behavior

- **Cold start:** {what happens when the app launches from killed — splash, restore last route, route from deep link / notification payload, auth check}
- **Warm start:** {what happens when the app resumes from background — biometric re-auth if timeout exceeded, data refresh policy}
- **Backgrounded:** {default behavior — timers paused, uploads continued via background tasks, location updates behavior}
- **Killed:** {state-restoration strategy — none | deep-link-only | full restoration}

(Per-screen deltas from the default live in the relevant blueprint entry's **Lifecycle behavior** field.)

---

## Accessibility

(Omit this section entirely only if accessibility is explicitly descoped for this phase. Mobile accessibility is mandatory by default.)

- **Semantic labels:** {VoiceOver / TalkBack labeling rule for controls, images, decorative elements}
- **Focus order:** {rule for screen focus traversal — logical / top-to-bottom; handling of offscreen content}
- **Dynamic type:** {rule — honor user font-size setting; layout reflow policy}
- **Switch control / external keyboard:** {rule — all interactive elements reachable}
- **Accessibility shortcuts:** {system-level shortcuts the app surfaces — e.g., Rotor custom actions on iOS, TalkBack gestures on Android}
- **Platform-specific commitments:** {iOS VoiceOver specifics; Android TalkBack specifics}

---

## Global States

| Condition | What the user sees | Recovery path |
|-----------|--------------------|---------------|
| Offline | {description — e.g., "offline banner across affected screens; cached content if available; destructive actions disabled"} | {action} |
| Update required (force upgrade) | {description} | {action — link to store} |
| Maintenance mode | {description} | {action} |
| Feature-flag gated | {description} | {action} |
| App-version deprecated (soft) | {description} | {action} |

---

## Orientation & Form Factor

- **Orientation default:** {portrait-only | portrait-and-landscape — with per-screen exceptions}
- **Per-screen orientation overrides:** {list exceptions, or "None — orientation policy uniform across screens"}
- **Tablet / iPad strategy:** {optimized split-view | stretch-only | not supported}
- **Foldable / large-screen strategy:** {supported behavior, or "Not optimized."}
- **Landscape considerations:** {rule — e.g., "video playback and camera only"}

---

## App Store Presence

(Omit this section entirely if store presence is owned elsewhere or not yet in scope.)

- **Store listing surfaces:** {screenshots per device class, preview video, subtitle, keywords}
- **What's-new strategy:** {release-note cadence and audience}
- **Review-prompt strategy:** {when the in-app review prompt is triggered — referenced by name from Shared Surfaces}

---

## Home-Screen Widgets & Lock-Screen

(Omit this section entirely if no widgets are in scope.)

- **Widgets offered:** {list — kind, surface, size classes}
- **Content shown per widget:** {role-level, not literal}
- **Refresh cadence:** {rule per widget}
- **Deep-link targets:** {which screens widgets route to on tap}
- **Lock-screen presence:** {iOS lock-screen widgets / live activities, Android lock-screen notifications — or "None."}

---

## App Intents / Siri Shortcuts / Google Assistant

(Omit this section entirely if no voice-invoked actions are in scope. Full voice UIs belong in `VOICE_IA.md`.)

- **Exposed intents:** {list — name, parameters, invocation phrases described by role}
- **Routing:** {which screen each intent lands on, and what is pre-filled}
- **Authentication requirement per intent:** {rule}

---

## Wearable Companions

(Omit this section entirely if no wearable surfaces are in scope.)

- **Platforms:** {watchOS | Wear OS | both}
- **Functionality subset surfaced:** {list — the wearable is not a miniature app; state what it does}
- **Sync strategy with phone app:** {rule}

---

## CarPlay / Android Auto

(Omit this section entirely if CarPlay / Auto are not supported.)

- **Functionality subset surfaced:** {list}
- **UI template used:** {CarPlay template family, Android Auto equivalent}

---

## Internationalization

(Omit this section entirely if single-locale.)

- **Locales supported:** {list}
- **Locale detection:** {rule — OS locale, in-app override}
- **RTL support:** {rule — layout mirroring policy, or "No RTL locales in scope."}
- **Date / number / currency formatting:** {rule}

---

## Telemetry

(Omit this section entirely if telemetry is not in scope for this document.)

- **Screen-view event naming:** {convention — e.g., `screen_viewed` with `screen` property}
- **Key-action event names:** {per screen or shared — high-level only, not parameter schemas}
- **Privacy contract:** {what the app promises not to send}

---

## Traceability Matrix

### Use cases → screens

| Use case | Screen(s) | Notes |
|----------|-----------|-------|
| {UC-ID} | `{ScreenName}`, `{ScreenName}` | {brief note — e.g., "Primary surface"; if sibling IAs exist, note other channels} |
| {UC-ID} | — | No mobile surface — {one-line reason}. |

(Every use case from the input document appears here.)

### Screens → use cases

| Screen | Use case(s) | Classification if none |
|--------|-------------|------------------------|
| `{ScreenName}` | {UC-IDs} | — |
| `{ScreenName}` | — | chrome / nav-container / auth-gate / error / splash / onboarding-slide / informational |

(Every screen from the inventory appears here.)

---

## Relationship to Other Documents

- **Product proposal and use cases:** This document realizes use cases on the native-mobile surface. Use-case IDs in the traceability matrix match the source use cases document.
- **Per-unit SPECs:** SPECs for mobile units must reference the MOBILE_IA entry for the screen(s) or shared surface(s) they implement, and use screen names, section roles, permission names, and deep-link patterns from this document verbatim.
- **CONTRACT_REGISTRY.md** (if present): per-screen data dependencies may reference endpoint names; wire-format shapes are owned by the contract registry, not redefined here.
- **WEB_IA.md / CLI_IA.md / VOICE_IA.md / TUI_IA.md** (if present): parallel IA documents for other surfaces. Use-case traceability must be consistent across all IAs — every use case surfaces on at least one channel or is explicitly classified as having none.
- **DESIGN.md / platform HIG and Material compliance:** visual design decisions (colors, typography, spacing, animation curves, iconography) are owned downstream of this document, not here.

---

## Open Questions

- [ ] {Question — e.g., "Should the notification-permission prompt fire at cold start for first-time users, or be deferred to the first feature that benefits from notifications?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Full screen-template inventory with navigation position, platform availability, deep-link status, and audience classification
- Per-screen blueprint: purpose, sections, primary CTA, data dependencies, gestures, orientation, permissions, platform variations, full state matrix (including offline and permission-denied), entry / exit points, lifecycle behavior, parameter-dependent behavior
- Platform strategy (iOS / Android / both; native vs cross-platform; minimum OS versions; feature-parity rules)
- Navigation model (root container, per-tab stacks, modal conventions, back semantics, state restoration)
- Deep-link strategy (URL scheme, universal / App Links, auth-gated behavior, destination coverage)
- Permissions strategy (what, when, rationale, denial handling)
- Multi-screen flow descriptions
- Shared native-surface inventory (alerts, sheets, toasts, share sheets, pickers, biometric prompts, pull-to-refresh, skeletons, empty-states)
- Push-notification architecture (categories, triggers, routing, rich media, platform differences)
- Lifecycle behavior (cold / warm / background / killed)
- Global states (offline, update-required, maintenance, feature-flag, deprecation)
- Orientation and form-factor strategy (portrait / landscape, phone / tablet / foldable)
- Accessibility commitments (VoiceOver / TalkBack, focus, dynamic type, switch control)
- Bidirectional traceability between use cases and screens, cross-referenced with sibling IAs when present
- Conditional sections (App Store presence, widgets, App Intents, wearable, CarPlay/Auto, i18n, telemetry) when applicable to the product

### Out of scope

- Implementation details — view models, state machines, DI wiring, navigation-library specifics, function signatures — owned by per-unit `SPEC.md`
- Visual design — colors, typography, spacing, animation curves, iconography, illustration selection, platform-theme tokens — owned by DESIGN.md / HIG / Material compliance at execution time
- Final UX copy and microcopy — owned by a UX-writing phase
- WebView-wrapped experiences — owned by `WEB_IA.md`
- Terminal-invoked binaries — owned by `CLI_IA.md`
- Full voice UIs — owned by `VOICE_IA.md` (App Intents / Siri Shortcuts / Google Assistant as mobile-app invocation points are in scope here)
- Desktop TUI apps — owned by `TUI_IA.md`
- Backend / API wire formats — owned by `CONTRACT_REGISTRY.md`
- App signing, code signing, distribution pipelines, store submission automation — owned by DEPLOYMENT docs
- Persona research and user research — owned by a separate research phase
- Database schema, business logic, algorithm design — owned by architecture and unit SPECs

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `platforms`, `screens_documented`, `flows_documented`, `shared_surfaces`, `deep_linkable_screens`, `permissions_requested`, `push_categories`, `use_cases_covered`, `use_cases_total`, `auth_screens`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Every screen template in the inventory has a full per-screen blueprint entry below
- [ ] Every per-screen entry declares screen identifier, navigation position, platform availability, deep-link status, auth requirement, sections, primary CTA (or explicit chrome / nav-container / auth-gate / error / splash / onboarding-slide / informational classification), data dependencies, gestures, orientation, permissions involved, platform variations (or "identical"), full state matrix (including offline and permission-denied), entry points, exit points, lifecycle behavior, parameter-dependent behavior, shared surfaces invoked
- [ ] Every screen with parameters documents parameter-dependent behavior (at minimum: not-found and unauthorized branches for screens targeting a named resource)
- [ ] Navigation model is stated once; every screen's navigation position references it
- [ ] Deep-link strategy is stated once; every deep-linkable screen appears in the destination coverage and conforms to the scheme; every screen declares "not deep-linkable" if it is not
- [ ] Permissions strategy is stated once; every per-screen permission reference uses a name from the Permissions Strategy table
- [ ] Platform variations are documented for every screen, or "identical across platforms" is stated explicitly
- [ ] Push Notifications section is present (or the entire app declares no notifications in Identity & Context and this section is omitted)
- [ ] Accessibility section is present and non-trivial (or accessibility is explicitly descoped in Identity & Context)
- [ ] Every use case in the input use cases document maps to at least one screen in the traceability matrix, or is explicitly listed as having no mobile surface with a one-line reason
- [ ] Every screen in the inventory is referenced by at least one use case in the traceability matrix, or is explicitly classified (chrome, nav-container, auth-gate, error, splash, onboarding-slide, informational)
- [ ] Every flow step references an existing screen template from the inventory (no flow introduces new screens)
- [ ] Shared surfaces appear only in the dedicated Shared Surfaces section, not scattered in per-screen entries
- [ ] Conditional sections (App Store presence, widgets, App Intents, wearable, CarPlay/Auto, i18n, telemetry) are either fully completed or fully omitted — never present-and-empty
- [ ] No implementation detail (view-model classes, state-flow types, DI wiring, function signatures, navigation-library APIs) appears anywhere in the document
- [ ] No visual-design decisions (colors, typography tokens, spacing values, animation curves, specific iconography) appear anywhere in the document
- [ ] No final UX copy — section and action roles are described, not literal strings
- [ ] Frontmatter `platforms`, `use_cases_covered`, and `use_cases_total` are present; any traceability gap is explained in the Traceability Matrix
- [ ] If sibling IAs (`WEB_IA.md`, `CLI_IA.md`, `VOICE_IA.md`, `TUI_IA.md`) are inputs, the traceability matrix cross-references use cases spanning multiple channels
