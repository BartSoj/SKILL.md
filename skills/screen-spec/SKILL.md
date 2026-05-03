---
name: SCREEN_SPEC.md
description: Produce a deep per-screen design specification extending one MOBILE_IA.md blueprint — screen shell, region layout, section composition, per-component layout detail, action inventory, gesture and interaction specifications, state-specific layouts, orientation and form-factor rules, platform variations, screen-specific motion, and accessibility commitments. Use when asked to create a SCREEN_SPEC.md for a screen, write a per-screen layout spec, deepen a MOBILE_IA blueprint with composition and gesture detail, pin a flagship or gesture-rich screen for implementation, or produce a SCREEN_SPEC.md.
---

# Task: Create SCREEN_SPEC.md for a Mobile Screen

## Objective

Produce a SCREEN_SPEC.md that serves as the single source of truth for composing one screen of a native mobile application: the screen shell (status bar, nav bar, bottom area, safe-area rules, scroll behavior), the region layout, the per-section composition with spacing and container treatment cited by token name, the per-component layout for non-standard widgets, an action-inventory table with placement and visibility rules, per-gesture interaction specs (target, effect, conflict resolution, haptics, cancel behavior), state-specific layout deltas, orientation and tablet rules, platform-variation deltas (iOS / Android / iPadOS), screen-specific motion, and accessibility commitments. The defining claim — and the definition of "done" — is that **the gap between MOBILE_IA's blueprint plus DESIGN.md plus this SCREEN_SPEC and the screen as built is purely mechanical**: an implementing agent reading those three artifacts composes the screen on each target platform without inventing spatial decisions, gesture handling, or platform-variant treatment.

The commonest violation is silently restating something MOBILE_IA already owns — re-listing sections, re-declaring the primary CTA, re-enumerating gestures, re-stating platform availability, redefining a state from the state matrix — instead of extending it with composition and interaction detail. SCREEN_SPEC owns *how each named region and section is composed and how each gesture behaves*; MOBILE_IA owns *that those regions, sections, and gestures exist*. When the SCREEN_SPEC needs a fact that is missing from MOBILE_IA (e.g., a section the screen has but MOBILE_IA does not list, a gesture not declared in MOBILE_IA's blueprint), surface that gap as an open question for `/MOBILE_IA.md` to resolve — do not silently add it.

---

## Inputs

1. **Screen identifier** (required) — the stable screen ID from MOBILE_IA.md's Screen Inventory. The skill operates on exactly one screen per invocation.
2. **MOBILE_IA.md** (required) — the parent IA hub. Provides the per-screen blueprint to extend: route or deep-link, purpose, audience, auth requirement, platform set (iOS / Android / iPadOS), sections, primary CTA, secondary actions, gestures listed, orientation support, permissions involved, state matrix (including offline and permission-denied), entry / exit points, lifecycle behavior, parameter-dependent behavior, shared surfaces invoked. SCREEN_SPEC reads the entry for the named screen end-to-end, plus the centralized Navigation Model, Deep-Link Strategy, Permissions Strategy, and Shared Surfaces sections it references.
3. **DESIGN.md** (required if present) — global design system: design tokens (spacing, color, typography, elevation, radius), component library (with variants), layout templates (e.g., `mobile-stack`, `mobile-tabs`, `mobile-modal`, `mobile-sheet`, `mobile-detail`), breakpoints and safe-area rules, motion principles, typography scale, touch-target minimums. SCREEN_SPEC cites tokens, components, and templates by name from this document — it does not redefine them.
4. **Sibling SCREEN_SPEC.md files** (optional, auto-discovered) — SCREEN_SPECs for screens this screen flows into or out of, when MOBILE_IA's flows reference them. Discovery mechanism: walk the entry / exit points and flow references in MOBILE_IA's per-screen blueprint and load any sibling `mobile-screens/<screen-id>/SCREEN_SPEC.md` that exists. Use these only to keep transition direction and shared surface invocation consistent — do not duplicate their content.
5. **Prior `mobile-screens/<screen-id>/SCREEN_SPEC.md`** (optional, auto-discovered — only if refining) — read fully; preserve every resolved open question and every decision that survives the new MOBILE_IA. Re-run citations against the current MOBILE_IA and DESIGN — do not assume stale section names, token names, component names, layout template names, or gesture names are still valid.

Read-set size: MOBILE_IA + DESIGN are always read; 0–2 sibling SCREEN_SPECs and an optional prior version. Read MOBILE_IA's per-screen entry end-to-end — partial reads cause silent drift between blueprint and spec.

---

## Rules

### Selection rules

SCREEN_SPEC is opt-in. Before writing, verify the screen meets at least one criterion below; if it meets none, the screen should stay with its inline blueprint in MOBILE_IA.md and this skill should not produce output.

- Composition exceeds ~30 lines if expressed inline in MOBILE_IA's per-screen entry.
- Two or more states in MOBILE_IA's state matrix have materially different layouts (not just skeleton-vs-content).
- The screen uses gesture combinations not covered by DESIGN.md's component-level gestures.
- Platform behavior diverges substantively between iOS and Android in spatial composition (not just chrome).
- Tablet / iPad uses a composition different from phone.
- Multiple work units touch this screen and need a shared composition contract.
- The screen is flagship or high-traffic and the team has decided to invest in deep specification.

If the screen meets no criterion, surface the situation as the only entry in section 13 Open Questions with a recommendation to keep the screen inline in MOBILE_IA, rather than producing a half-spec.

### Hub-leaf boundary rules

MOBILE_IA owns identity; SCREEN_SPEC owns composition and interaction detail. Each fact lives in exactly one of them.

- **Sections list, primary CTA, gestures-listed, platform set, data dependencies, traceability, lifecycle behavior, parameter-dependent behavior, shared-surfaces-invoked, state-matrix rows** are owned by MOBILE_IA. Reference them by name; do not redefine them in SCREEN_SPEC.
- **Spatial composition of each section, container treatment, spacing tokens, action placement, gesture target / effect / conflict-resolution / haptics, per-state layout deltas, platform composition variations, screen-specific motion, accessibility-driven layout reflow** are owned by SCREEN_SPEC. They are not in MOBILE_IA.
- If the SCREEN_SPEC needs to invoke a section, gesture, state, or shared surface not declared in MOBILE_IA's blueprint, surface the gap as an open question for `/MOBILE_IA.md`. Do not silently add it.
- Section names are copied **verbatim** from MOBILE_IA's per-screen entry. Renaming, splitting, or merging sections is a MOBILE_IA edit, not a SCREEN_SPEC edit.

### Cross-reference rules

Every quantity, component, template, gesture, state, platform, and surface in SCREEN_SPEC must resolve to a name owned by MOBILE_IA, DESIGN, or this skill's canonical vocabulary.

- **Design tokens** are cited by token name (`space-12`, `radius-md`, `elevation-card`, `motion-medium-out`), never by raw values (`48dp`, `8px`, `200ms`). If a needed token does not exist in DESIGN, surface it as an open question for `/DESIGN.md`.
- **Components** are cited by component name from DESIGN's component library, with the variant if applicable (e.g., `Button.primary`, `Cell.inset-grouped`, `Sheet.bottom`).
- **Layout template** in section 1 is cited by name from DESIGN's `§ Page templates` (e.g., `mobile-stack`, `mobile-tabs`, `mobile-modal`, `mobile-sheet`, `mobile-detail`). If the screen uses a one-off composition not in the template inventory, write a one-line custom override and surface as an open question for `/DESIGN.md` if the override should be promoted to a template.
- **Gestures** are cited by canonical name matching MOBILE_IA's vocabulary: `tap`, `double-tap`, `long-press`, `swipe-left`, `swipe-right`, `swipe-up`, `swipe-down`, `pull-to-refresh`, `pinch`, `drag-to-reorder`, `edge-swipe-back`, `swipe-to-dismiss`. New gesture names require a MOBILE_IA edit first.
- **States** in section 8 use the exact state names from MOBILE_IA's state matrix for this screen (`Default / loaded with data`, `Loading (cold start)`, `Empty`, `Error — transient network`, `Offline`, `Permission-denied — {permission name}`, `Backgrounded (mid-screen)`, etc.). Do not invent new states.
- **Platforms** are cited as `iOS`, `Android`, `iPadOS` — exact casing. The platform set in section 1 must be a subset of MOBILE_IA's `Platform availability` for this screen.
- **Shared surfaces** referenced in actions or gestures are cited by name from MOBILE_IA's Shared Surfaces section. Per-screen inline redefinitions are forbidden (Rule analog of MOBILE_IA Rule 13).
- **Permissions** referenced in state-specific layouts are cited by name from MOBILE_IA's Permissions Strategy.

### Completeness rules

- Every region in section 3 must declare position, dimensions or sizing rule, background / border / separator treatment, internal alignment / scroll behavior, sticky-vs-fixed classification, and z-index relative to other regions.
- Every section in section 4 must declare order in the screen, container treatment, internal layout, spacing above / below / between siblings (by token name), header treatment, and per-section sub-states (loading skeleton, error inline, empty).
- Section 5 entries are required only where a section uses non-standard composition — form layouts, list-item compositions, action-group layouts, custom widgets. Sections that are a single component invocation do not need a section-5 entry.
- Section 6 must enumerate **every** interactive control on the screen across all states — nav-bar actions, tab-bar items, primary CTA, FAB, inline buttons, swipe-action affordances, long-press menus, cell tap targets. Each row declares position, component variant, visibility rule, and disabled rule.
- Section 7 must enumerate **every** gesture supported on this screen with target, effect, conflict resolution against native back gesture and tab-bar gestures, haptic feedback type and trigger, and cancel behavior.
- Section 8 must address **every** state from MOBILE_IA's state matrix for this screen. States whose layout is identical to `Default / loaded with data` are listed with "Layout identical to default; only chrome and inline messages differ." — silent omission is forbidden.
- Section 10 must either enumerate per-platform divergences or read "All platforms render identically per the cross-platform component set; no platform variations." — silent equivalence is forbidden.
- Section 11 must either enumerate screen-specific motion or read "Inherits global Motion defaults from DESIGN.md `§ Motion`; no screen-specific motion." — silent omission is forbidden.

### Precision rules

- Use exact names — section names from MOBILE_IA, region names, action labels (placeholder form, since literal copy belongs to MOBILE_IA), token names, component names, gesture names, state names, platform names. No "appropriate", "relevant", "as needed", "etc."
- Describe behavior as ordered or imperative declarations ("Pull-to-refresh on the content region triggers a refetch and yields haptic `medium` at the trigger threshold; cancel before threshold dismisses without refetch."), not as vague summaries ("supports refresh appropriately").
- When two valid names exist (e.g., `Sheet.bottom` vs `BottomSheet`), use the name that DESIGN uses verbatim. If DESIGN is silent or inconsistent, surface as an open question for `/DESIGN.md`.
- Distinguish "hidden" from "not rendered": a hidden control occupies layout space; a not-rendered control does not. State which.

### Single YAML frontmatter block

- Exactly one YAML frontmatter block at the top of the SCREEN_SPEC.md output, never two — even when refining an existing SCREEN_SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `screen_id`, `platforms`, `template`, `sections_specified`, `actions_specified`, `gestures_specified`, `states_specified`, `platform_variations`, `open_questions`).
- Every count in the frontmatter matches the body exactly: `sections_specified` is the number of section entries in § 4; `actions_specified` is the number of rows in § 6's table; `gestures_specified` is the number of gesture entries in § 7; `states_specified` is the number of state entries in § 8; `platform_variations` is the number of per-platform divergence entries in § 10 (zero when § 10 reads the "no platform variations" sentence); `open_questions` is the number of unresolved questions in § 13 (zero when § 13 reads "All questions resolved.").
- `platforms` is the platform set from § 1 as an array (`[ios, android]`, `[ios, android, ipados]`, `[ios]`, etc.) — exact lowercase tokens.
- `template` is the layout-template name from § 1.
- `status` is `complete` only when § 13 reads "All questions resolved."; `has_open_questions` when one or more questions remain unresolved; `blocked` only when a missing input (e.g., a section the screen has but MOBILE_IA does not list, a needed token absent from DESIGN) prevents authoring the SCREEN_SPEC and the gap is explicit in § 13.

### Forbidden patterns

- **Defining new design tokens, components, or layout templates.** SCREEN_SPEC consumes DESIGN's vocabulary; it does not extend it.
- **Specifying brand decisions** — color palettes, typography choices, illustration style, iconography selection. Those belong in DESIGN.md.
- **Final user-facing copy.** MOBILE_IA owns labels at role level; final copy is owned by a UX-writing phase. SCREEN_SPEC may use short disambiguating placeholders (`[Fork]`) but no full microcopy.
- **Code or framework-specific syntax.** No SwiftUI / Compose / React Native / Flutter source. No view-model classes, state-flow types, navigation-library route builders. Those belong in per-unit SPECs.
- **Overlapping concerns with MOBILE_IA.** Do not redefine the sections list, primary CTA, gestures-listed-in-blueprint, platform availability, data dependencies, lifecycle behavior, parameter-dependent behavior, traceability to use cases, or shared-surfaces-invoked. Reference them; do not restate them.
- **Inventing new state, gesture, or section names.** All such names come from MOBILE_IA or this skill's canonical gesture vocabulary. Anything new is an open question first.
- **Pseudocode for layout composition.** Composition is described in prose with token / component / template citations, not as code-like layout DSLs.

### No-code rule

- The SCREEN_SPEC must not contain code or pseudocode. If a non-obvious composition cannot be described unambiguously in prose with token and component citations, surface it as an open question for `/DESIGN.md` so the missing component / template is added there — do not work around the gap with code.

---

## Output Format

The SCREEN_SPEC.md file must follow this exact structure. Every section is mandatory. If a section has no content (e.g., no per-component layout entries beyond standard composition), include the heading with the explicit empty-state sentence prescribed below — do not drop the heading.

```markdown
---
skill: SCREEN_SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
screen_id: {screen-id-from-MOBILE_IA}
platforms: [{ios, android, ipados — subset that applies}]
template: {layout-template-name | custom-override}
sections_specified: {N}
actions_specified: {N}
gestures_specified: {N}
states_specified: {N}
platform_variations: {N}
open_questions: {N}
---

# SCREEN_SPEC: {ScreenName} — {short name}

## 1. Identity & Cross-references

- **Screen ID:** `{screen-id}` from MOBILE_IA.md
- **Route or deep-link:** `{copied verbatim from MOBILE_IA's per-screen entry}`
- **Layout template:** `{template-name from DESIGN.md § Page templates}` (or one-line custom override with rationale)
- **Platforms targeted:** {iOS | Android | iPadOS — subset of MOBILE_IA's Platform availability for this screen}
- **Design tokens cited:** {comma-separated list of every token referenced anywhere in this SCREEN_SPEC — `space-12`, `radius-md`, `elevation-card`, `motion-medium-out`, …}
- **Component dependencies:** {comma-separated list of every DESIGN component referenced — `Button.primary`, `Cell.inset-grouped`, `Sheet.bottom`, …}
- **Sibling screens (in flows):** {SCREEN_SPEC IDs of screens this screen flows into or out of, per MOBILE_IA's entry/exit points — or "None — terminal screen in its flow."}

---

## 2. Screen Shell

- **Status bar treatment:** {visible | hidden | blended}; {light-content | dark-content | adaptive}
- **Navigation bar:** {large title | standard | hidden}; back-button behavior — {default pop | custom dismiss | hidden}; right-side actions — {list of nav-bar actions, by role}
- **Bottom area:** {tab bar | action bar | none}; safe-area treatment — {edge-to-edge with bottom inset | inset content area | inherits app default}
- **Safe-area insets:**
  - **Top:** {rule — e.g., "content begins below status bar; nav bar absorbs top inset"}
  - **Bottom:** {rule — e.g., "primary CTA inset by `space-16` above bottom safe area; tab bar absorbs bottom inset"}
  - **Leading / Trailing:** {rule — e.g., "content edges align to `space-16` from leading and trailing safe-area edges"}
- **Scroll behavior:** {scrolls | static | parallax-header collapse | inset-header pinned}; bounces {yes | no}; supports `pull-to-refresh` {yes | no — reference Shared Surfaces}
- **Vertical rhythm baseline:** {token name — e.g., "`space-8` baseline grid; section gaps use `space-24`"}

---

## 3. Region Layout

For each named region — top bar, content, sidebar / drawer (iPadOS or large screens), bottom action area, tab bar, FAB — provide a sub-block. Omit regions the screen does not have; write "{region} — not present on this screen." for any region the reader might expect.

### Top bar

- **Position and dimensions:** {fixed top; height — token name; spans full width}
- **Background / border / separator:** {token names from DESIGN}
- **Internal alignment / scroll behavior:** {static | collapses with content scroll | parallax}
- **Sticky vs fixed:** {sticky | fixed | scrolls-away}
- **Z-index relative to other regions:** {integer or token name}

### Content

- **Position and dimensions:** {fills between top bar and bottom area}
- **Background / border / separator:** {tokens}
- **Internal alignment / scroll behavior:** {single-axis vertical scroll | horizontal pager | grid | list}
- **Sticky vs fixed:** {scrolls}
- **Z-index relative to other regions:** {integer or token name}

(Repeat for each region. Use the same five-bullet structure.)

---

## 4. Section Composition

Iterate every section listed in MOBILE_IA's per-screen entry for this screen, in the same order, with section names copied verbatim.

### `{Section name from MOBILE_IA}`

- **Order in screen:** {1, 2, 3, …}
- **Container treatment:** {card | inline | inset-grouped | plain-grouped | full-bleed}
- **Internal layout:** {description — e.g., "vertical stack of rows; each row a `Cell.inset-grouped` with leading icon, title, trailing chevron"}
- **Spacing above:** `{token name}`
- **Spacing below:** `{token name}`
- **Spacing between siblings (within section):** `{token name}`
- **Header treatment:** {none | inline title `text-section-header` | floating title with `space-12` below | sticky}
- **Per-section sub-states:**
  - **Loading skeleton:** {description — e.g., "three placeholder `Cell` rows with shimmer per `motion-shimmer-loop`"}
  - **Error inline:** {description — e.g., "`InlineError` row replaces the section content; CTA `[Retry]`"}
  - **Empty:** {description — e.g., "`EmptyState.compact` with neutral icon and role-level prompt"}

(Repeat for every section. Section count must match `sections_specified` in frontmatter.)

---

## 5. Per-Component Layout Detail

(For sections whose composition is a single standard component invocation, no entry is needed. Provide entries only where a section uses non-standard composition — form layout, list-item composition, action-group layout, custom widget.)

### `{Component or composite within section X}`

- **Composition:** {description — e.g., "two-column form; labels in the leading column at `space-160` width, fields in the trailing column flexing to remaining width"}
- **Internal spacing:** {token names — e.g., "field-to-field gap `space-12`; label-to-field gap `space-8`"}
- **Component variants used:** {DESIGN component names — e.g., `TextField.outlined`, `Stepper`, `Toggle.switch`}
- **Variant-by-content rule:** {if applicable — e.g., "destructive actions render `Button.destructive`; neutral actions render `Button.tertiary`"}

(Repeat for each non-standard composition. If none: "All sections use standard component compositions per DESIGN.md; no per-component layout detail required.")

---

## 6. Action Inventory & Placement

| Action (role) | Position | Component variant | Visibility rule | Disabled rule |
|---|---|---|---|---|
| {role description, not literal copy — e.g., "Fork the resource"} | {nav-bar trailing | content area at index N | bottom action area | FAB anchored bottom-trailing | swipe-action on cell row | long-press menu on cell} | {DESIGN component name} | {when this control appears — e.g., "always" | "only when viewer is not the owner" | "only in `Default / loaded with data` state"} | {when this control is enabled — e.g., "always enabled" | "disabled while a request is in flight" | "disabled when offline"} |

(Add one row per interactive control. Include every control reachable on this screen across all states. Action count must match `actions_specified` in frontmatter.)

---

## 7. Gesture & Interaction Specifications

For each gesture listed in MOBILE_IA's per-screen blueprint plus any gesture introduced by an action in section 6, provide an entry. Gesture names must come from the canonical vocabulary in the Cross-reference rules.

### `{gesture-name}` on `{target region or element}`

- **Effect:** {what happens — e.g., "fetches latest activity; updates the Activity section in place"}
- **Conflict resolution:** {how this gesture coexists with `edge-swipe-back` and tab-bar gestures — e.g., "horizontal swipe within the tab strip is captured for tab switching; outside the strip, `edge-swipe-back` takes precedence"}
- **Haptic feedback:** {`light` | `medium` | `heavy` | `selection` | `success` | `warning` | `error` | `none`} at {trigger point — e.g., "the moment the refresh threshold is crossed"}
- **Cancel behavior:** {what happens if the gesture is canceled mid-stream — e.g., "release before threshold dismisses without refetch; in-flight network request is canceled"}

(Repeat per gesture. Gesture count must match `gestures_specified` in frontmatter.)

---

## 8. State-Specific Layouts

For every state in MOBILE_IA's state matrix for this screen, declare what changes in the layout. States whose layout is identical to `Default / loaded with data` use the prescribed identical-layout sentence.

### `{State name from MOBILE_IA}`

- **Layout deltas:** {what changes vs `Default / loaded with data` — e.g., "Content region replaced by `EmptyState.full` centered vertically; primary CTA in bottom action area becomes `[Create]`"} *(or "Layout identical to default; only chrome and inline messages differ.")*
- **Transition behavior:** {how the layout enters this state — e.g., "skeleton fades out per `motion-fast-out` and content fades in per `motion-medium-in`"}

(Repeat for every state. State count must match `states_specified` in frontmatter.)

---

## 9. Orientation & Form Factor

- **Portrait:** {default behavior — e.g., "default composition as described above"}
- **Landscape:** {supported | not supported}; if supported — {what reflows — e.g., "two-column layout with section list leading and detail trailing; content scrolls within trailing column"}
- **iPad / tablet:** {not supported | stretches without reflow | optimized — describe split-view, primary / secondary column composition, multitasking behavior}
- **Foldables / split-screen:** {adaptation rule — e.g., "in compact width on a foldable, falls back to phone composition; in expanded width, uses tablet composition"} *(or "Not optimized; falls through to phone composition.")*

---

## 10. Platform Variations

(If the screen renders identically on all platforms in section 1's `Platforms targeted` list, replace the per-platform entries with: "All platforms render identically per the cross-platform component set; no platform variations." Otherwise enumerate every divergence.)

### iOS-specific behavior

- {composition or interaction divergence — e.g., "overflow action sheet renders as iOS sheet (`Sheet.ios`) anchored to the trailing nav-bar action"}
- {platform-idiom — e.g., "large title in nav bar collapses to standard on scroll"}
- {action sheet vs alert — e.g., "destructive confirmation uses iOS action sheet"}

### Android-specific behavior

- {composition or interaction divergence — e.g., "overflow action sheet renders as Material `Sheet.bottom`"}
- {hardware-back handling — e.g., "hardware back dismisses the bottom sheet first, then pops the stack"}
- {FAB convention — e.g., "primary CTA renders as Material `FAB.extended` anchored bottom-trailing"}

### iPadOS-specific behavior

(Omit entirely if iPadOS is not in section 1's `Platforms targeted`.)

- {split-view behavior — e.g., "primary column hosts the section list; secondary column hosts the detail content"}
- {multitasking — e.g., "supports Slide Over and Split View; minimum width matches the phone composition"}

---

## 11. Motion & Transitions

- **Screen-entry transition:** {push | modal-present | sheet-present | crossfade | none}; cite motion token from DESIGN if applicable.
- **Screen-exit transition:** {push-pop | modal-dismiss | sheet-dismiss | crossfade | none}.
- **Element transitions within the screen:** {list-row insertion / removal / reorder; tab-content swap; section expand / collapse — by token name}
- **Loading-state choreography:** {sequence — e.g., "skeleton appears immediately; content fades in per `motion-medium-in` once data resolves; minimum visible duration `motion-min-skeleton`"}
- **Toast / snackbar appearance:** {position, dismissal — reference Shared Surfaces by name}

(If no screen-specific motion: "Inherits global Motion defaults from DESIGN.md `§ Motion`; no screen-specific motion.")

---

## 12. Accessibility

- **VoiceOver / TalkBack labels:** {where labels deviate from on-screen text — e.g., "icon-only nav-bar trailing action announces the action role"}
- **Dynamic Type / font scaling:** {what reflows when type scales up; cap at largest accessibility size — e.g., "rows reflow to two lines at extra-large; truncates at the largest accessibility size"}
- **Reduced motion:** {what motion is suppressed — e.g., "skeleton shimmer disabled; element transitions become crossfades"}
- **Increased contrast:** {what changes — e.g., "separator weight increases; background tint deepens per DESIGN.md tokens"}
- **Hit-target sizing:** {minimums per region — cite DESIGN.md `§ Touch targets`; e.g., "all interactive cells meet `touch-target-min`"}
- **Focus order (external keyboard / switch control):** {logical order — e.g., "1. nav-bar leading, 2. nav-bar trailing actions, 3. content sections in order, 4. bottom action area"}

---

## 13. Open Questions

This section must be EMPTY before implementation begins. If any questions remain unresolved, the SCREEN_SPEC is not ready. The user will resolve open questions after reviewing the spec — do not ask during generation.

When drafting, you will encounter decisions where multiple approaches are defensible and the input documents do not clearly favor one. Do NOT silently pick an answer to keep this section empty. Instead:

- If a question has one obviously correct answer given MOBILE_IA + DESIGN, resolve it yourself and write the decision into the appropriate section.
- If a question has no clear answer — multiple valid approaches exist, or MOBILE_IA / DESIGN are ambiguous or silent — list it here with proposed options and tradeoffs.

Format for open questions:

- [ ] {Question}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {if you lean one way, say so and why — but leave the decision to the user}

Common gaps to surface here rather than resolve silently:

- A section the screen has but MOBILE_IA's blueprint does not list → resolve in `/MOBILE_IA.md` first.
- A gesture not declared in MOBILE_IA's gestures list → resolve in `/MOBILE_IA.md` first.
- A token, component, or layout template needed but missing from DESIGN → resolve in `/DESIGN.md` first.
- A platform-specific composition for a platform not in MOBILE_IA's Platform availability → resolve in `/MOBILE_IA.md` first.

Once the user resolves all questions, move each resolution into the appropriate section and replace this section with:

"All questions resolved."
```

---

## Scope

### In scope

- Identity and cross-reference block (screen ID, route, layout template, platforms targeted, design tokens cited, component dependencies, sibling screens)
- Screen shell (status bar, navigation bar, bottom area, safe-area insets, scroll behavior, vertical rhythm baseline)
- Region layout (top bar, content, sidebar / drawer, bottom action area, tab bar, FAB) with position, treatment, scroll behavior, sticky-vs-fixed, z-index
- Section composition for every section named in MOBILE_IA's blueprint — order, container treatment, internal layout, spacing tokens, header treatment, per-section sub-states
- Per-component layout detail for non-standard compositions (forms, list items, action groups, custom widgets)
- Action inventory and placement table — every interactive control with position, component variant, visibility rule, disabled rule
- Gesture and interaction specifications — per gesture: target, effect, conflict resolution, haptics, cancel behavior
- State-specific layout deltas for every state in MOBILE_IA's state matrix
- Orientation and form-factor rules (portrait, landscape, iPad / tablet, foldables, split-screen)
- Platform-variation deltas (iOS, Android, iPadOS) for spatial composition and idiomatic interaction
- Screen-specific motion and transitions (entry, exit, intra-screen element transitions, loading choreography, toast appearance)
- Accessibility commitments (VoiceOver / TalkBack labels, dynamic type, reduced motion, increased contrast, hit-target sizing, focus order)
- Open questions surfacing genuine ambiguity in MOBILE_IA, DESIGN, or sibling SCREEN_SPECs

### Out of scope

- Identity-level facts about the screen — sections list, primary CTA, gestures listed, platform set, data dependencies, traceability, lifecycle behavior, parameter-dependent behavior, shared-surfaces-invoked, state-matrix membership — owned by `MOBILE_IA.md`
- Design tokens, components, layout templates, motion principles, typography scale, touch-target minimums — owned by `DESIGN.md`
- Final user-facing copy and microcopy — owned by a UX-writing phase
- Implementation code or framework syntax — view models, state-flow types, navigation-library APIs, function signatures — owned by per-unit `SPEC.md`
- HTTP wire formats and endpoint contracts — owned by `INTERFACES.md`
- Cross-screen flows and global navigation rules — owned by `MOBILE_IA.md`'s Flows and Navigation Model
- Push-notification routing, lifecycle defaults, global states — owned by `MOBILE_IA.md`
- Screens whose composition fits inline in MOBILE_IA's blueprint — those do not need a SCREEN_SPEC at all (see Selection rules)
- Web pages, CLI commands, voice intents, TUI views — owned by `WEB_IA.md`, `CLI_IA.md`, `VOICE_IA.md`, `TUI_IA.md`

---

## Quality Checklist

Before considering a SCREEN_SPEC.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `screen_id`, `platforms`, `template`, `sections_specified`, `actions_specified`, `gestures_specified`, `states_specified`, `platform_variations`, `open_questions`)
- [ ] Frontmatter has exactly one YAML block at the top of the file — never two
- [ ] Frontmatter counts match the body exactly (`sections_specified` = entries in § 4, `actions_specified` = rows in § 6, `gestures_specified` = entries in § 7, `states_specified` = entries in § 8, `platform_variations` = per-platform divergence entries in § 10, `open_questions` = unresolved items in § 13)
- [ ] `status` is `complete` if § 13 reads "All questions resolved." and `has_open_questions` otherwise (use `blocked` only when a missing input prevents authoring and § 13 makes the gap explicit)
- [ ] `screen_id` matches a screen in MOBILE_IA.md's Screen Inventory exactly
- [ ] `platforms` is a subset of MOBILE_IA's `Platform availability` for this screen, with exact lowercase tokens (`ios`, `android`, `ipados`)
- [ ] `template` is a name from DESIGN.md `§ Page templates` (or a one-line custom override is documented in § 1 with rationale)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty / "All questions resolved." or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable alongside MOBILE_IA's per-screen entry and DESIGN, without opening sibling SCREEN_SPECs
- [ ] § 1 lists every design token referenced anywhere in the document, every DESIGN component referenced, the layout template, and the platform set
- [ ] Every section heading from MOBILE_IA's per-screen blueprint for this screen appears verbatim in § 4
- [ ] No section in § 4 is invented relative to MOBILE_IA's blueprint — gaps are surfaced in § 13
- [ ] Every region in § 3 declares position / dimensions, background / border / separator, internal alignment / scroll behavior, sticky-vs-fixed, z-index
- [ ] Every interactive control on the screen across all states appears as a row in § 6
- [ ] Every gesture in § 7 uses a canonical name from the Cross-reference rules vocabulary (or matches MOBILE_IA's gestures-listed verbatim)
- [ ] Every gesture in § 7 declares target, effect, conflict resolution, haptics, cancel behavior
- [ ] Every state in MOBILE_IA's state matrix for this screen appears in § 8 — states with identical layouts are still listed with the prescribed identical-layout sentence
- [ ] § 9 declares portrait, landscape, iPad / tablet, and foldables / split-screen behavior — silent omission is forbidden
- [ ] § 10 either enumerates per-platform divergences for every platform in § 1 or contains the prescribed "no platform variations" sentence
- [ ] § 11 either enumerates screen-specific motion or contains the prescribed inherits-defaults sentence
- [ ] § 12 declares VoiceOver / TalkBack, dynamic type, reduced motion, increased contrast, hit-target sizing, and focus order
- [ ] No design tokens are defined in this document — all tokens are cited by name from DESIGN.md
- [ ] No components or layout templates are defined in this document — all are cited by name from DESIGN.md
- [ ] No raw values (px, dp, ms, hex, rgb) appear anywhere — only token names
- [ ] No final user-facing copy appears — actions and labels are described by role, with short disambiguating placeholders allowed (e.g., `[Fork]`)
- [ ] No code or pseudocode appears anywhere in the document
- [ ] No facts owned by MOBILE_IA are restated — sections list, primary CTA, gestures-listed-in-blueprint, platform availability, data dependencies, lifecycle behavior, parameter-dependent behavior, traceability, shared-surfaces-invoked, state-matrix membership are referenced, not redefined
- [ ] Section names in § 4 are copied verbatim from MOBILE_IA's per-screen entry
- [ ] Permissions referenced in § 8's permission-denied state rows use names from MOBILE_IA's Permissions Strategy
- [ ] Shared surfaces referenced in § 6, § 7, or § 11 use names from MOBILE_IA's Shared Surfaces section
- [ ] If the screen meets none of the Selection rules criteria, § 13 contains the single recommendation to keep the screen inline in MOBILE_IA, and `status` is `has_open_questions`
