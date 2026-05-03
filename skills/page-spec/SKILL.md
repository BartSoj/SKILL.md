---
name: PAGE_SPEC.md
description: Produce a deep, per-page composition specification for a single web page — page shell, region layout, section composition, per-component layout detail, action inventory and placement, state-specific layouts, responsive behavior, page-specific motion, and accessibility notes — extending the lightweight per-page blueprint that already lives in WEB_IA.md by citing DESIGN.md tokens, components, and layout templates without redefining them. Use when asked to create a PAGE_SPEC.md for a page, write a per-page deep design specification, fix a complex page's composition before implementation, extend a WEB_IA blueprint with full spatial detail, or produce a PAGE_SPEC.md.
---

# Task: Create PAGE_SPEC.md for a Single Web Page

## Objective

Produce a PAGE_SPEC.md that pins the full spatial composition of one web page: page shell, named regions, section ordering and grouping, per-component layout overrides, every interactive control with its placement and visibility rule, layout differences across states, breakpoint reflow, page-specific motion, and accessibility commitments beyond the global rules. The defining discipline — and the definition of "done" — is that **the gap between the design layer and the page is purely compositional**: an implementing agent reading WEB_IA.md's per-page blueprint, DESIGN.md's tokens and components, and this PAGE_SPEC.md composes the page exactly as designed without inventing any spatial decision (token literals, region anchors, button order, breakpoint reflow, focus order). PAGE_SPEC is a Tier 2 leaf artifact: WEB_IA.md owns *identity* (the page exists, has these named sections, has these states); PAGE_SPEC.md owns *composition* (where everything sits, how it groups, how it reflows).

The commonest violation is silently inventing a token literal, a component variant, or a layout template name that does not exist in DESIGN.md. When DESIGN.md is silent, ambiguous, or contradictory on a question the page must answer — a token for a specific spacing, a component for a specific affordance, a breakpoint name — the question belongs in § 12 Open Questions with options, tradeoffs, and a recommendation. Do not invent. The PAGE_SPEC must never introduce a design decision that is not already traceable to DESIGN.md (tokens, components, templates, breakpoints, motion principles) or WEB_IA.md (sections, primary CTA, states, data dependencies, traceability).

---

## Inputs

1. **Page identifier** (required) — the stable ID from WEB_IA.md (e.g., `B5`, `C1`). The skill operates on exactly one page per invocation. If the supplied identifier does not exist in WEB_IA.md's per-page blueprints, set `status: blocked` and surface the gap in § 12.
2. **WEB_IA.md** (required) — the parent IA hub. Provides the per-page blueprint this PAGE_SPEC extends: route pattern, URL parameters, purpose, audience, auth requirement, referenced use cases, sections list, primary CTA, secondary actions, data dependencies, state matrix, entry/exit points, parameter-dependent behavior, and shared surfaces invoked. Read end-to-end — partial reads cause silent identity drift.
3. **DESIGN.md** (required if present) — the global design system: tokens (color, type, spacing, radii, elevation), component library, layout templates (named page shapes — `centered-narrow`, `workspace-three-pane`, `settings-rail`, `marketing-bleed`, `dashboard-stack`, `modal-dialog`), responsive breakpoints, and motion principles. The PAGE_SPEC cites these by name; it never redefines them. If DESIGN.md is absent, set `status: blocked` and surface the gap.
4. **Sibling PAGE_SPEC.md files referenced from this page's flows** (optional, auto-discovered) — when this page links into a multi-page flow declared in WEB_IA.md § Flows, sibling PAGE_SPECs may be read to keep transition behavior consistent. Discovery: walk the flows in WEB_IA.md that reference this page, then read each sibling page's `web-pages/<page-id>/PAGE_SPEC.md` if it exists. Read only the specific siblings cited.
5. **Prior `web-pages/<page-id>/PAGE_SPEC.md`** (optional, auto-discovered if refining) — preserve every resolved open question, every named region, every breakpoint reflow rule already calibrated. Re-run citations against the current DESIGN.md and WEB_IA.md — do not assume stale token names or section names are still valid.

Read-set size: 2–4 artifacts typically (page identifier is metadata, not a read; WEB_IA + DESIGN.md + optionally sibling specs + optionally prior PAGE_SPEC).

### When PAGE_SPEC creation is warranted

PAGE_SPEC is **opt-in**. Most pages stay with their inline Composition note in WEB_IA.md. Recommend a PAGE_SPEC when any of the following hold:

- Composition would exceed ~30 lines if expressed inline in WEB_IA's per-page entry
- The page has multiple states whose layouts differ materially (not just skeleton vs. content swap)
- The page uses interaction patterns not covered by DESIGN.md's component library (multi-select trees, drag-drop, virtualized scrolling, custom data viz)
- The page has page-specific motion or choreography beyond DESIGN.md's global motion rules
- Multiple work units touch the same page and need a stable shared composition reference
- The page is flagship or high-traffic, where the cost of redesign is high

If none of these hold and the user is asking for a PAGE_SPEC anyway, produce the document but state in § 1 under a brief preamble line that "Inline composition in WEB_IA would have sufficed for this page" — do not refuse to produce the artifact.

---

## Workflow

PAGE_SPEC generation proceeds in five phases. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Read & Reconcile

Load WEB_IA.md in full. Locate the per-page blueprint for the supplied page identifier. Extract every fact this PAGE_SPEC must respect verbatim: route pattern, section names, primary CTA, secondary actions, state matrix entries, shared surfaces invoked. Any divergence between WEB_IA and a draft PAGE_SPEC is a WEB_IA win — fix the draft.

Load DESIGN.md in full. Index the available token IDs (color, type, spacing, radii, elevation), component names and variants, layout template names, breakpoint names, and motion-principle names. The PAGE_SPEC may cite only what this index contains.

If the page is part of a flow with sibling PAGE_SPECs, read those siblings' § 9 Interaction & Motion to keep handoff behavior consistent.

If a prior PAGE_SPEC exists for this page, load it and preserve resolved decisions; re-run citations against current DESIGN.md and WEB_IA.md.

### Phase 2: Pick Layout Template

Choose the layout template by name from DESIGN.md `§ Page templates`. The chosen template constrains the rest of the spec — page shell dimensions, region anchors, default content alignment all come from the template. If no DESIGN.md template fits the page's needs, declare a one-line override under § 1 (e.g., "Custom shape — settings-rail base with an extra rail on the right for live preview") and document the override in § 12 as an open question for DESIGN.md to absorb if it recurs.

### Phase 3: Compose Top-Down

Walk the document outside-in: page shell → regions → sections within regions → per-component overrides within sections. At each level, cite tokens by name (never literal values), components by DESIGN.md name, and section names verbatim from WEB_IA.md.

### Phase 4: Action Inventory, States, Responsive, Motion, Accessibility

Build the actions table in § 6 by walking WEB_IA's primary CTA + secondary actions + shared surfaces invoked, then add page-specific actions (per-row context menus, inline affordances). Every interactive control on the page must appear as a row.

For § 7, walk WEB_IA's state matrix and document only the states whose layout differs from the default loaded composition. States with identical composition collapse into a single line.

For § 8, walk DESIGN.md's breakpoints and document reflow per breakpoint. Touch-target rule cites DESIGN.md `§ Touch targets` if present.

For § 9, capture only motion that is page-specific. Motion already covered by DESIGN.md `§ Motion` is referenced by name, not restated.

For § 10, document only accessibility commitments that are page-specific. Global rules from DESIGN.md are referenced, not restated.

### Phase 5: Validate & Finalize

Run the Quality Checklist. Update frontmatter counts to reflect the final body exactly. Set `status` based on § 12.

---

## Rules

### Citation discipline

- Cite design tokens by **name**, never by literal value (`space-12`, not `48px`; `border-subtle`, not `#E0E0E0`).
- Cite components by **name and variant** from DESIGN.md (`Button (primary, lg)`, not `<button class="btn-primary">` and not just `Button`).
- Cite the layout template by **name** from DESIGN.md `§ Page templates`, not by re-describing geometry.
- Cite section names **verbatim** from WEB_IA.md's per-page entry — do not rename, abbreviate, or pluralize.
- Cite breakpoints by **token name** from DESIGN.md (`breakpoint-md`, not `768px` or `tablet`).
- Cite sibling pages by their **WEB_IA ID** (e.g., `B5`, `C1`).
- Cite use cases by their **WEB_IA ID** when relevant; the PAGE_SPEC does not re-list use cases.

### Single source of truth — boundary with WEB_IA

WEB_IA owns *identity*: the page exists, has these named sections, has this primary CTA, has these states, has these data dependencies, has these entry/exit points, realizes these use cases. PAGE_SPEC does not redefine any of these. Specifically:

- Do not redefine the route pattern or URL parameters.
- Do not rename, add, or remove sections from WEB_IA's section list.
- Do not change the primary CTA.
- Do not invent new states beyond WEB_IA's state matrix.
- Do not redeclare data dependencies (page-level).
- Do not re-list use cases or entry/exit points.

If the PAGE_SPEC reveals a gap or error in WEB_IA, surface it in § 12 — do not silently patch it in the PAGE_SPEC.

### Single source of truth — boundary with DESIGN.md

DESIGN.md owns the design system. PAGE_SPEC does not:

- Define new tokens (those go in DESIGN.md).
- Define new components or component variants (those go in DESIGN.md).
- Specify brand decisions — color choices, font choices, iconography selection, imagery direction (those belong to DESIGN.md).
- Override DESIGN.md's global motion rules without justification (cite DESIGN.md `§ Motion`; only deviate when § 9 explicitly notes a page-specific override and the override is named in § 12).

If the PAGE_SPEC needs a token, component, or template that DESIGN.md does not provide, surface it in § 12 — do not invent.

### No final copy

PAGE_SPEC owns composition, not labels. Do not write final button labels, headings, or microcopy. Describe the role (`primary CTA for repository creation`) rather than the literal string. One short placeholder label (e.g., `[Create]`) is acceptable for table-cell disambiguation — no more.

### No code or framework syntax

- No code or pseudocode. PAGE_SPEC is a design contract, not an implementation plan.
- No framework-specific syntax. No JSX, no Vue templates, no Tailwind utility class names, no CSS selectors. Composition is described in prose anchored to DESIGN.md citations.

### Strict-template completeness

Every section in the Output Format template is mandatory. If a section has no applicable content, the heading is present with an explicit `Not applicable — {one-line reason}.` line. Silent omission is forbidden.

### Single YAML frontmatter block

Exactly one YAML frontmatter block at the top of the PAGE_SPEC.md output, never two — even when refining an existing PAGE_SPEC, merge into a single block. The block carries every field enumerated in the Output Format template (`skill`, `date`, `status`, `page_id`, `template`, `sections_specified`, `actions_specified`, `states_specified`, `breakpoints_specified`, `open_questions`).

Counts must match the body exactly:
- `sections_specified` = number of section blocks in § 4
- `actions_specified` = number of rows in § 6's table
- `states_specified` = number of state-specific layout blocks in § 7 (states collapsed under "rendering with default composition" do not count)
- `breakpoints_specified` = number of breakpoint blocks in § 8
- `open_questions` = number of unresolved questions in § 12 (zero when § 12 reads "All questions resolved.")

`status` is `complete` only when § 12 reads "All questions resolved."; `has_open_questions` when one or more questions remain unresolved; `blocked` only when a missing input (the page is not in WEB_IA, DESIGN.md is missing, a needed token does not exist) prevents authoring and the gap is explicit in § 12.

### Precision over vagueness

No "appropriate", "relevant", "as needed", "etc.". Use exact token names, component variants, region anchors, and breakpoint names. If you cannot be exact, surface the question in § 12 rather than hide behind a vague word.

---

## Output Format

The PAGE_SPEC.md file must follow this exact structure. Every section is mandatory. If a section has no applicable content, include the heading with `Not applicable — {one-line reason}.` underneath.

```markdown
---
skill: PAGE_SPEC.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
page_id: {page-id-from-WEB_IA}
template: {layout-template-name-from-DESIGN}
sections_specified: {N}
actions_specified: {N}
states_specified: {N}
breakpoints_specified: {N}
open_questions: {N}
---

# PAGE_SPEC: {page_id} — {page name from WEB_IA}

## 1. Identity & Cross-references

- **Page ID:** `{page-id}` (from WEB_IA.md)
- **Route / URL pattern:** `{route}` (verbatim from WEB_IA.md — single source of truth check)
- **Layout template:** `{template-name-from-DESIGN}` *(or "Custom — {one-line override description}")*
- **Design tokens cited:** `{token-id}`, `{token-id}`, … *(comma-separated; every token referenced anywhere in this PAGE_SPEC must appear here)*
- **Component dependencies:** `{Component (variant)}`, `{Component (variant)}`, … *(every component composed by this page)*
- **Sibling pages:** `{page-id}`, `{page-id}` *(WEB_IA IDs for pages this page hands off to or receives from in flows; or "None.")*

## 2. Page Shell

{Outer dimensions, max-width, alignment. Page-level chrome that frames the page (top bar, breadcrumbs, footer): what is present, sticky/fixed behavior. Vertical rhythm baseline. Cite tokens by name.}

## 3. Region Layout

For each named region the layout template defines (header, sub-nav, content, sidebars, footer, action zone, …):

### {Region name}

- **Position and dimensions:** {anchors — left/right/top/bottom; min/max widths cited by token name}
- **Background, border, separators:** {tokens by name}
- **Internal alignment / scroll behavior:** {rule}
- **Sticky / fixed / absolute classification:** {sticky | fixed | absolute | static}
- **Z-index relative to other regions:** {ordering rule, by region name}

(Repeat for every region in the layout template. If the page suppresses a region the template normally provides, state the suppression and reason.)

## 4. Section Composition

For each section listed in WEB_IA.md's per-page entry — preserve names verbatim:

### {Section name}

- **Order and grouping:** {which other sections share a visual region with this one; or "Stands alone."}
- **Container treatment:** {card | inline | bordered | unstyled — cite tokens for radius / border / elevation}
- **Internal layout:** {single column | two-column | grid; density rule cited by token name}
- **Spacing above / below / between siblings:** {token names}
- **Header treatment:** {heading + description | heading only | prefix icon | none}
- **Empty / loading / error sub-states of this section specifically:** {what changes within this section's region; or "None — section uses default DESIGN.md sub-states."}

(Repeat for every section in WEB_IA's section list. Do not add, remove, or rename sections.)

## 5. Per-Component Layout Detail

Where a section uses non-standard component composition, describe the override:

### {Section name} — {composition concern}

- **Form layout:** label position, input size, helper-text placement, spacing between fields (token names)
- **Action group layout:** order of buttons, alignment, separation rule, dirty-state visibility
- **Data display layout:** table density, row interactions, hover affordance
- **Custom widget:** full spatial spec

(Repeat for each non-default composition. If everything composes from DESIGN.md defaults: "All sections compose from default DESIGN.md component layouts; no per-component overrides.")

## 6. Action Inventory & Placement

Every interactive control on the page, including those inherited from WEB_IA's primary CTA, secondary actions, and shared surfaces invoked, plus page-specific actions (per-row menus, inline affordances).

| Action | Position | Component variant | Visibility rule | Disabled rule |
|--------|----------|-------------------|-----------------|---------------|
| {role description, no final copy} | {region + anchor} | `{Component (variant)}` | {always | when {condition}} | {never | when {condition}} |

(Every primary CTA, secondary action, and shared-surface trigger from WEB_IA must appear. Add rows for page-specific affordances. If the page is chrome/informational with no actions: "Not applicable — page has no interactive controls.")

## 7. State-Specific Layouts

For each state from WEB_IA's state matrix whose layout differs materially from the default loaded composition:

### {State name}

- **What changes:** {skeleton positions, error banner placement, empty-state illustration anchor, redirect destination — by region and section name, with token citations}
- **Transition behavior into this state:** {instant | fade | slide — duration token name}

(Repeat for each state with a differing layout.)

**States rendering with default composition:** {comma-separated list — e.g., "Loading (subsequent), Loaded with data."}

(If every state shares the default composition: "All states from WEB_IA render with default composition.")

## 8. Responsive Behavior

Per breakpoint from DESIGN.md, what reflows. Default composition is the widest declared breakpoint; smaller breakpoints describe deltas.

### ≥ `{breakpoint-largest}` (default)

{Reference the composition documented in §§ 3–5; no restatement needed.}

### `{breakpoint-mid}`–`{breakpoint-largest}`

- **What reflows:** {by region and section name, with token citations}

### < `{breakpoint-smallest}`

- **What collapses:** {regions that become drawer/sheet/stack; sections that hide; nav patterns that change}

**Touch-target sizing rule:** {cite DESIGN.md `§ Touch targets`, or "Not applicable — DESIGN.md does not define touch-target rules."}

(Add or remove breakpoint blocks to match the breakpoints DESIGN.md defines.)

## 9. Interaction & Motion

Page-specific motion not already covered in DESIGN.md `§ Motion`:

- **Element transitions:** {saving spinner → check-mark, dirty-state pulse, etc. — duration and easing by DESIGN.md token name}
- **Choreography:** {when an action triggers multiple visual changes, the order and timing}
- **Page-entry / page-exit transitions:** {if any, cited by DESIGN.md token; or "None — uses DESIGN.md default page transitions."}
- **Toast / shared-surface appearance position and duration:** {only if differing from DESIGN.md defaults}

(If everything follows DESIGN.md defaults: "Not applicable — page uses only DESIGN.md global motion principles.")

## 10. Accessibility Notes

Page-specific accessibility commitments beyond DESIGN.md global rules:

- **Tab order:** {ordered list of regions and key controls}
- **ARIA roles and landmarks:** {sub-nav as `tablist`, form regions as `role="region"` with named labels, etc.}
- **Live regions:** {for dynamic content — save status, async results}
- **Focus management:** {when modal opens/closes, when route changes within the page, when sub-section navigates}
- **Screen-reader-only text:** {clarifying labels needed beyond visible copy}

(If global DESIGN.md rules cover everything this page needs: "Not applicable — page relies on DESIGN.md global accessibility rules.")

## 11. Implementation Notes

(Optional — omit the section entirely if there are no notes.)

- {Existing component to reuse and where it lives}
- {Known pattern this page should follow, with reference}
- {Anti-pattern to avoid for this specific page, with reason}

## 12. Open Questions

This section must be EMPTY before implementation begins on units that consume this PAGE_SPEC. If any questions remain unresolved, the spec is not ready. Resolve silently when one answer is clearly correct given DESIGN.md + WEB_IA.md; surface only when multiple valid approaches exist or inputs are ambiguous.

Format:

- [ ] {Question — e.g., "DESIGN.md provides `space-8` and `space-12` but not `space-10`; the section header gap reads as wanting an intermediate value."}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- One page per invocation, identified by stable WEB_IA ID
- Page shell, region layout, section composition, per-component layout overrides for one page
- Action inventory and placement for every interactive control on the page
- State-specific layouts for states whose composition differs from the default
- Responsive reflow per DESIGN.md breakpoint
- Page-specific motion and choreography that extend DESIGN.md global motion rules
- Page-specific accessibility commitments that extend DESIGN.md global rules
- Citations of DESIGN.md tokens, components, and templates by name only
- Cross-references to WEB_IA.md sections, primary CTA, states, and sibling pages

### Out of scope

- Page identity (route, URL parameters, sections list, primary CTA, state matrix, data dependencies, entry/exit points, use-case traceability) — owned by WEB_IA.md
- Design system — tokens, components, layout templates, breakpoints, global motion, global accessibility rules — owned by DESIGN.md
- Brand decisions — color choices, font choices, iconography, imagery direction — owned by DESIGN.md
- Final user-facing copy — labels, headings, microcopy — owned by a UX-writing phase
- Component implementation — props, hooks, state management, framework code — owned by per-unit `SPEC.md`
- Backend wire formats and data shapes — owned by `INTERFACES.md` (or the contract registry of the project)
- Multi-page flow definitions — owned by WEB_IA.md `§ Flows`; PAGE_SPEC may cite a sibling page's hand-off but does not redefine the flow
- Pages that meet none of the "When PAGE_SPEC is warranted" criteria — those stay with inline composition in WEB_IA.md
- Multiple pages — exactly one page per PAGE_SPEC invocation

---

## Quality Checklist

Before considering a PAGE_SPEC.md complete, verify:

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `page_id`, `template`, `sections_specified`, `actions_specified`, `states_specified`, `breakpoints_specified`, `open_questions`)
- [ ] Frontmatter counts match the body exactly: `sections_specified` = section blocks in § 4, `actions_specified` = rows in § 6's table, `states_specified` = state-specific layout blocks in § 7 (states under "rendering with default composition" do not count), `breakpoints_specified` = breakpoint blocks in § 8, `open_questions` = unresolved items in § 12
- [ ] `status` is `complete` if § 12 reads "All questions resolved."; `has_open_questions` if any remain; `blocked` only when a missing input prevents authoring and § 12 makes the gap explicit
- [ ] `page_id` matches a per-page blueprint that exists in WEB_IA.md
- [ ] § 1 Route / URL pattern matches WEB_IA.md verbatim
- [ ] § 1 Layout template is a name from DESIGN.md `§ Page templates` (or a one-line override declared explicitly)
- [ ] § 1 Design tokens cited list contains every token referenced anywhere in §§ 2–10
- [ ] § 1 Component dependencies list contains every component referenced anywhere in §§ 3–6
- [ ] § 4 contains a section block for every section in WEB_IA.md's per-page entry, with names verbatim — no additions, no renames, no removals
- [ ] § 6 actions table contains a row for every primary CTA, secondary action, and shared-surface trigger from WEB_IA, plus rows for page-specific affordances; rows describe roles, not final copy
- [ ] § 7 covers every state from WEB_IA's state matrix — either as a state-specific block or in the "rendering with default composition" line
- [ ] § 8 contains one block per breakpoint that DESIGN.md defines (matching breakpoint count)
- [ ] Every token in the document is cited by name (e.g., `space-12`), never as a literal value (e.g., `48px`)
- [ ] Every component in the document is cited by DESIGN.md name and variant (e.g., `Button (primary, lg)`)
- [ ] Every breakpoint is cited by DESIGN.md token name (e.g., `breakpoint-md`)
- [ ] Every section name in § 4 and § 5 matches WEB_IA.md verbatim
- [ ] Every sibling page reference uses the WEB_IA ID format (e.g., `B5`)
- [ ] No new design tokens, components, or layout templates are defined in the PAGE_SPEC; gaps are surfaced in § 12 instead
- [ ] No final user-facing copy appears (labels described by role, not literal string; one short `[placeholder]` per action row is acceptable)
- [ ] No code, pseudocode, JSX, Vue, Tailwind class names, or CSS selectors appear anywhere
- [ ] No content from WEB_IA is restated — route, sections, primary CTA, states, data dependencies, entry/exit points, traceability are referenced, not redefined
- [ ] Conditional sections (§ 11 Implementation Notes) are either populated or omitted entirely — never present-and-empty
- [ ] Mandatory sections (§§ 1–10, § 12) are present; sections with no applicable content carry an explicit `Not applicable — {reason}.` line
- [ ] § 12 Open Questions contains only genuine ambiguities — not questions resolvable from DESIGN.md + WEB_IA.md
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Output is self-contained — readable and actionable in conjunction with WEB_IA.md and DESIGN.md, without opening other PAGE_SPECs unless they are explicitly cited as siblings
