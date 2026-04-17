---
name: WEB_IA.md
description: Generate a web information architecture registry — page inventory, URL strategy, navigation, per-page blueprints, flows, and use-case traceability. Use when asked to create a web IA, design the site map, produce a page inventory, define URL strategy and navigation, or produce a WEB_IA.md.
---

# Task: Generate WEB_IA.md — Web Information Architecture Registry

## Objective

Produce a WEB_IA.md that serves as the single source of truth for a web application's information architecture: the full page-template inventory, URL strategy, global navigation, per-page blueprint (purpose, sections, primary CTA, data dependencies, state matrix, entry/exit points), cross-page flows, shared UI surfaces, and bidirectional traceability from product use cases to pages. An agent or designer reading this document knows exactly what pages exist, what lives on each page, how users move between them, and how every use case is realized on the web surface — without making any independent architectural or content decisions. WEB_IA sits between product requirements (proposal, use cases) and per-unit web SPECs, the same way CONTRACT_REGISTRY sits between architecture and per-unit backend SPECs.

---

## Inputs

1. **Product / proposal document** (required) — product purpose, target users, and rationale. Sets the scope and audience frame for the IA.
2. **Use cases document** (required) — enumerates user interactions and jobs-to-be-done. Primary input. Every use case with a web surface must map to one or more pages in the inventory.
3. **Architecture document** (optional but recommended) — tech-stack constraints (SPA vs MPA, SSR/SSG/CSR, routing library, auth model). Informs URL strategy and state boundaries. If absent, infer conservative defaults and flag non-obvious choices as open questions.
4. **Existing web codebase** (auto-discovered) — discover routes, pages, and shared components from source (e.g., route files, `pages/` or `app/` directories, router config). Treat as reference only — WEB_IA is the target state, not a mirror of the current state.
5. **Decision documents** (optional) — architectural decisions that constrain the IA (e.g., "all write actions require confirmation modal", "workspace context is selected via URL segment").
6. **Design constraints document** (optional) — brand/DESIGN.md/impeccable config. Read only to pick up IA-affecting constraints (e.g., mandated navigation pattern). Do not pull visual-design details in.
7. **Contract registry** (optional) — if present, per-page data dependencies may reference endpoints by name.

The user may override scope (e.g., "only the authenticated app area, not marketing site"). Support that by recording the narrowed scope in the Identity & Context section and excluding out-of-scope surfaces from the inventory.

---

## Workflow

Web IA generation proceeds in seven phases: scope, URL and navigation, inventory, per-page blueprint, flows and shared surfaces, global states and responsive, traceability and validation. Phases are sequential — later phases depend on earlier decisions — but revisit earlier phases if a later phase reveals a gap.

### Phase 1: Scope & Audience

Derive from the product doc and use cases document the target web surface (marketing site, authenticated app, admin back-office, embedded widget, or some combination) and the audiences the IA serves. Record explicitly what is *not* in scope for this WEB_IA document. Keep audience treatment thin — named user types and their 3–7 top tasks per audience. No deep persona work.

### Phase 2: URL Strategy & Global Navigation

Decide routing conventions and global chrome before enumerating pages. This prevents per-page drift where each page invents its own URL shape or navigation.

Decide:
- **Casing** (kebab-case vs camelCase) and trailing-slash policy
- **Query params vs path segments** for filters, pagination, sorting, and tab selection
- **Deep-link semantics** for modal, filter, and drawer states (are modals URL-addressable?)
- **Reserved prefixes** (e.g., `/api`, `/_next`, `/assets`)
- **Authenticated-vs-anonymous variants** of the primary navigation (header, footer, sidebar, mobile drawer)
- **Navigation patterns in use** — tabs, breadcrumbs, command palette, in-page section nav

Use architecture-doc hints where available; fall back to conservative defaults. Surface non-obvious choices as open questions rather than guessing.

### Phase 3: Page Inventory

From use cases and architecture, enumerate the full set of **page templates** (not page instances — see Rule 1). Classify each: authenticated vs public, audience served, chrome/nav/auth/error/content category. Build a single sitemap table that makes the hierarchy visible (either via indentation or grouped subsections).

For a new product, derive pages from use cases: every use case with a web surface implies at least one page where the user completes or progresses that use case. For an existing codebase, discover current routes and reconcile against use cases — adding missing pages, flagging orphan pages.

### Phase 4: Per-Page Blueprint

Iterate every page template in the inventory and produce its full entry. This is the bulk of the output. Each entry must include every mandatory field listed in the Output Format section: route pattern, URL parameters, name, purpose, audience, auth requirement, referenced use cases, sections/blocks, primary CTA, secondary actions, data dependencies, state matrix, entry points, exit points, and parameter-dependent behavior.

Work through pages in the order they appear in the sitemap. When writing sections/blocks, describe each region by **role and content**, not by component name or visual appearance (see Rule 6).

### Phase 5: Flows & Shared Surfaces

Identify multi-step user journeys — signup, onboarding, checkout, password reset, setup wizards, multi-page forms. For each, write a flow entry: ordered steps pointing to pages in the inventory, entry conditions, exit conditions, abort behavior, and the use case(s) the flow implements. Flows do not introduce new pages (Rule 11); they sequence existing ones.

Identify shared UI surfaces that cross pages: modals/dialogs, drawers, toasts/notifications, skeleton/loading patterns, empty-state patterns, and persistent global affordances (command palette, feature-flag banners, cookie banner, help widget, in-app tour). Consolidate these in the dedicated Shared UI Surfaces section (Rule 12). Per-page entries may list which shared surfaces they can invoke, but must not re-describe them.

### Phase 6: Global States & Responsive Strategy

Document global patterns independent of any single page: 404, 500, offline, unauthorized (403), unauthenticated (401), maintenance mode, feature-flag gates. One pattern per row — what page or surface the user sees, what actions are available.

Document IA-level responsive behavior: which navigation collapses to a drawer on narrow viewports, which secondary panels become modals on mobile, what is hidden or reordered by viewport class. Not pixel breakpoints — behavior rules.

### Phase 7: Traceability & Validation

Build the bidirectional use-case ↔ page matrix. Every use case in the input use cases document maps to at least one page, or is explicitly listed as having no web surface with a one-line reason. Every page in the inventory is referenced by at least one use case, or is explicitly classified (chrome, auth, error, informational).

Verify every rule holds: state matrix complete per page, primary CTA declared (or chrome classification), URL strategy respected across all routes, no component-level detail, no visual-design decisions, conditional sections either complete or absent. Update frontmatter counts to reflect the final document.

---

## Rules

These rules govern the output document. Violations are detected by the quality checklist.

### 1. Page templates, not page instances

A dynamic route like `/{owner}/{name}` is one page template, not N pages. Document each route pattern once. Note the URL parameters and enumerate parameter-dependent behavior (e.g., "if resource visibility is private and viewer lacks access → 404 not 403 to avoid leaking existence"). Do not enumerate concrete URL instances.

### 2. Primary CTA is mandatory per non-chrome page

Every page must declare **exactly one** primary CTA, or at most two with explicit justification, or be explicitly classified as a chrome / navigation / informational / error page with no CTA. A page that cannot name its primary CTA is confused — either split it or re-examine its purpose.

### 3. State matrix is mandatory per page

Every per-page blueprint must enumerate, at minimum: loading (first paint and subsequent navigations), loaded-with-data, empty (data set is empty), error (data fetch failure), unauthenticated (if auth-gated — redirect or soft gate?), unauthorized (authenticated but lacks permission), and not-found (for dynamic routes). For each state describe **what the user sees and what actions are available**. Do not describe visual appearance — colors, spacing, and typography are out of scope.

### 4. Every page maps to at least one use case or is explicitly classified

A page in the inventory either appears in the traceability matrix under at least one use case, or carries an explicit classification: chrome, auth, error, or marketing/informational. Silent orphan pages are not allowed.

### 5. Every use case maps to at least one page or is explicitly noted

Every use case in the input use cases document appears in the traceability matrix mapped to at least one page, or is listed as "no web surface" with a one-line reason (e.g., "CLI-only", "background job, no UI"). Silent drops are not allowed.

### 6. No component-level detail

Do not write prop types, hook signatures, state-management decisions, or component APIs. Those belong in per-unit SPECs. Describe sections by **role and content**, not by component name.

**Worked example — the IA-vs-SPEC line.** This is the single most important boundary in the document; internalize it.

> **In scope (WEB_IA):** "The resource detail page has a header with resource name, a secondary stat count, and a primary 'Fork' CTA; a navigation panel on the left; a primary content panel in the center showing the resource contents."
>
> **Out of scope (WEB_IA, belongs in per-unit SPEC):** "`<ResourceHeader>` takes `{resource: Resource, viewer: User}` props, returns a JSX tree containing…"

Prefer section-and-role descriptions over component-and-prop descriptions throughout the output.

### 7. No visual-design decisions

No colors, font families, exact spacing, component library choices, iconography, or imagery selections. Those belong in DESIGN.md / impeccable / the visual execution loop.

### 8. No final copy

Do not write final button labels, headings, or marketing copy. Describe the role ("primary CTA for forking the resource") rather than the literal string ("'Fork Repository'"). One short placeholder label (e.g., `[Fork]`) is acceptable for disambiguation — no more.

### 9. URL strategy is stated once, then enforced

The URL Strategy section is authoritative. Every route in the inventory must conform to the declared conventions (casing, trailing slash, query-vs-path rules, reserved prefixes). If an endpoint must deviate (e.g., external-standard compliance), document the exception inline in the blueprint entry and explain why.

### 10. Global navigation is stated once

The Global Navigation Architecture section is authoritative. Per-page blueprint entries reference the global nav by name — "standard authenticated header", "marketing footer" — and never re-describe it. If a page suppresses part of the global chrome (e.g., a focused onboarding step hides the top nav), note the suppression in the page entry.

### 11. Flows do not introduce new pages

Every flow step references an existing page template in the inventory. If a flow needs a step that has no corresponding page, add that page to the inventory first (with a full blueprint), then reference it from the flow.

### 12. Shared surfaces are consolidated

Modals, drawers, toasts, banners, and persistent global affordances live in the Shared UI Surfaces section — not scattered across per-page entries. Per-page entries may list which shared surfaces they can invoke by name; they must not redefine them.

### 13. Conditional sections must be omitted, not left empty

If Accessibility Commitments, SEO & Crawl, Internationalization, Telemetry Hooks, or Authenticated Context sections are not applicable for the project, omit them entirely. Do not leave them present with "N/A" content. Tight output is the house style (contract-registry is the reference).

### 14. Precision over vagueness

No "appropriate", "relevant", "as needed", "etc." Use exact names for pages, sections, CTAs, use case references, and route patterns. If you cannot be exact, flag it as an open question rather than hiding behind a placeholder word.

---

## Output Format

```markdown
---
skill: WEB_IA.md
date: {YYYY-MM-DD}
status: {complete | has_open_questions | blocked}
pages_documented: {N}
flows_documented: {N}
shared_surfaces: {N}
use_cases_covered: {N}
use_cases_total: {N}
auth_pages: {N}
public_pages: {N}
open_questions: {N}
---

# WEB_IA

> Single source of truth for the web application's information architecture.
> Every page template, URL pattern, navigation surface, flow, and shared UI
> surface is authoritative — per-unit web SPECs must reference these entries.

## Identity & Context

{One paragraph: product name, the slice of the web surface this document
 covers, and the audience frame. Reference the product proposal by path.
 State explicitly what surfaces are out of scope for this document.}

## Audience & Top Tasks

| Audience | Top tasks on the web |
|----------|---------------------|
| {audience name} | {3–7 comma-separated top tasks} |

(Repeat for each audience identified in the product doc.)

## URL Strategy

- **Casing:** {kebab-case | camelCase | mixed with rule}
- **Trailing slash:** {required | forbidden | either}
- **Query params vs path segments:** {rule — e.g., "filters and pagination in query params; resource identity in path segments"}
- **Deep-link semantics:** {modal / drawer / filter state URL-addressability rules}
- **Reserved prefixes:** {list, e.g., `/api`, `/_next`, `/assets`}
- **Exceptions:** {any routes that deviate, with reason — or "None."}

## Global Navigation Architecture

### Primary header
{Contents, authenticated-vs-anonymous variants, which pages suppress it.}

### Sidebar / secondary nav
{Contents, when present, which audiences see it.}

### Footer
{Contents, variants.}

### Mobile drawer
{What collapses into it, trigger affordance.}

### Navigation patterns in use
- {e.g., breadcrumbs on nested resource pages}
- {e.g., command palette globally}
- {e.g., tabs for within-page sub-sections}

## Page Inventory (Sitemap)

| Route pattern | Name | Auth | Audience | Use cases | Purpose |
|---------------|------|------|----------|-----------|---------|
| `/` | {name} | public | {audience} | {UC-IDs} | {one line} |
| `/{owner}/{name}` | {name} | {auth} | {audience} | {UC-IDs} | {one line} |

(Group by area — marketing, authenticated app, admin, etc. — via subsection
 headings or visible indentation. Repeat for every page template.)

---

## Per-Page Blueprints

### `{route pattern}` — {page name}

- **URL parameters:** {name: type — validation rule. Repeat for each. Or "None."}
- **Purpose:** {one sentence}
- **Audience:** {user types this page serves}
- **Auth requirement:** {public | authenticated | authenticated-with-permission: {permission name}}
- **Referenced use cases:** {UC-IDs, or explicit classification: chrome | auth | error | informational}

**Sections / blocks** (ordered, top-to-bottom / outside-in):

1. **{section name}** — {purpose and required content}
2. **{section name}** — {purpose and required content}

**Primary CTA:** {single action — e.g., "Submit resource creation"} *(or "No CTA — chrome/nav/informational page.")*

**Secondary actions:** {list, or "None."}

**Data dependencies:**
- {what the page needs to render — e.g., "resource metadata, viewer permissions, list of child items"}
- {if a contract registry exists: endpoint references by name}

**States:**

| State | What the user sees | Actions available |
|-------|-------------------|-------------------|
| Loading (first paint) | {description} | {actions} |
| Loading (subsequent) | {description} | {actions} |
| Loaded with data | {description} | {actions} |
| Empty | {description} | {actions} |
| Error | {description} | {actions} |
| Unauthenticated | {description or "N/A — public page"} | {actions} |
| Unauthorized | {description or "N/A — no permission gating"} | {actions} |
| Not found | {description or "N/A — static route"} | {actions} |

**Entry points:** {where users typically arrive from — links from other pages, navigation, deep links, external sources}

**Exit points:** {where users typically go next — primary CTA destination, secondary paths}

**Parameter-dependent behavior:** {how URL parameters change what is shown — e.g., "if resource visibility is private and viewer is not a member → 404 not 403". Or "N/A — static route."}

**Shared surfaces invoked:** {names from Shared UI Surfaces section, or "None."}

---

(Repeat the blueprint block for every page template in the inventory.)

---

## Flows

### {Flow name}

- **Purpose:** {one sentence}
- **Use cases implemented:** {UC-IDs}
- **Entry condition:** {what triggers the flow}
- **Steps:**
  1. `{route pattern}` — {what the user does here}
  2. `{route pattern}` — {what the user does here}
- **Success exit:** {destination page and state}
- **Abort behavior:** {can the user leave? what is preserved? where do they go?}
- **Resume on return:** {what happens if the user navigates away and returns mid-flow}

(Repeat for each flow. If none: "No multi-step flows in scope.")

---

## Shared UI Surfaces

| Surface | Kind | Trigger | Dismissal | Invoked by pages |
|---------|------|---------|-----------|------------------|
| {name} | {modal / drawer / toast / banner / command palette / tour} | {what opens it} | {how it closes} | {page names or "global"} |

(Repeat for every shared surface. If none: "No shared surfaces in scope.")

---

## Global State Patterns

| Condition | What the user sees | Recovery path |
|-----------|-------------------|---------------|
| 404 (unknown route) | {description} | {action} |
| 500 (server error) | {description} | {action} |
| Offline | {description} | {action} |
| Unauthorized (403) | {description} | {action} |
| Unauthenticated (401) | {description} | {action} |
| Maintenance mode | {description} | {action} |
| Feature flag gated | {description} | {action} |

---

## Responsive / Device Strategy

- **Narrow viewport:** {what collapses, what becomes a drawer/modal}
- **Wide viewport:** {what expands back, what appears alongside the primary panel}
- **Hidden or reordered by viewport:** {list or "None."}

(Behavior rules only — no pixel breakpoints.)

---

## Accessibility Commitments

(Omit this section entirely if accessibility is out of scope for this phase.)

- **Keyboard navigation:** {rule}
- **Skip links:** {rule}
- **Landmark structure:** {rule}
- **Focus management on route change:** {rule}

---

## SEO & Crawl

(Omit this section entirely if the product has no public pages.)

- **Indexable pages:** {which routes are public and crawlable}
- **Canonical URL strategy:** {rule}
- **Metadata source:** {per-page or template-level — where title/description come from}
- **sitemap.xml scope:** {which routes are included}

---

## Internationalization

(Omit this section entirely if the product is single-locale.)

- **Language routing pattern:** {e.g., `/{locale}/...` path segment | subdomain | Accept-Language}
- **Pages needing i18n:** {list or "all"}
- **RTL considerations:** {rule, or "No RTL languages in scope."}

---

## Telemetry Hooks

(Omit this section entirely if telemetry is not in scope for this document.)

- **Page-view event naming:** {convention — e.g., `page_viewed` with `route` property}
- **Key-action event names:** {per page or shared — high-level only, not parameter schemas}

---

## Authenticated Context

(Omit this section entirely if the app has no workspace/org/project scoping.)

- **Context selection:** {how users pick workspace/org/project}
- **URL reflection:** {how selected context appears in routes}
- **Navigation reflection:** {how context shows in global chrome}
- **Switch behavior:** {what happens when context changes — stay on page, redirect, reset?}

---

## Traceability Matrix

### Use cases → pages

| Use case | Page(s) | Notes |
|----------|---------|-------|
| {UC-ID} | `{route pattern}`, `{route pattern}` | {brief note, or "Primary surface."} |
| {UC-ID} | — | No web surface — {one-line reason}. |

(Every use case from the input document appears here.)

### Pages → use cases

| Page | Use case(s) | Classification if none |
|------|-------------|------------------------|
| `{route pattern}` | {UC-IDs} | — |
| `{route pattern}` | — | chrome / auth / error / informational |

(Every page from the inventory appears here.)

---

## Relationship to Other Documents

- **Product proposal and use cases:** This document realizes use cases on the web surface. Use case IDs in the traceability matrix match the source use cases document.
- **Per-unit SPECs:** SPECs for web units must reference the WEB_IA entry for the page(s) or shared surface(s) they implement, and use section-and-role names from this document.
- **CONTRACT_REGISTRY.md** (if present): per-page data dependencies may reference endpoint names; wire-format shapes are owned by the contract registry, not redefined here.
- **DESIGN.md / impeccable / visual execution loop:** visual design decisions (colors, typography, spacing, component library, imagery) are owned downstream of this document, not here.
- **CLI_IA.md** (future, if present): command-line surfaces are a parallel document, not part of this one.

---

## Open Questions

- [ ] {Question — e.g., "Should modal state be URL-addressable via query params, or stored only in memory?"}
  - **Option A:** {description} — {tradeoff}
  - **Option B:** {description} — {tradeoff}
  - **Recommendation:** {suggestion and reasoning}

(If none: "All questions resolved.")
```

---

## Scope

### In scope

- Full page-template inventory with URL patterns and sitemap hierarchy
- Per-page blueprint: purpose, sections, primary CTA, secondary actions, data dependencies, state matrix, entry/exit points, parameter-dependent behavior
- URL strategy and routing conventions for the web surface
- Global navigation architecture (header, footer, sidebar, mobile drawer, navigation patterns)
- Multi-step flow descriptions that sequence pages from the inventory
- Shared UI surface inventory (modals, drawers, toasts, banners, persistent global affordances)
- Global state patterns (404, 500, offline, unauthorized, unauthenticated, maintenance, feature flags)
- Responsive / device strategy at the IA (behavioral) level
- Bidirectional traceability between use cases and pages
- Conditional sections (accessibility, SEO, i18n, telemetry, authenticated context) when applicable to the product

### Out of scope

- Visual design — owned by DESIGN.md / impeccable / the visual execution loop
- Component-level implementation details (props, hooks, state management, component APIs) — owned by per-unit `SPEC.md`
- Final UX copy and microcopy — owned by a UX-writing phase
- Wireframes and mockups — produced separately in Figma / v0 / equivalent after the IA is settled
- Backend and API wire formats — owned by `CONTRACT_REGISTRY.md`
- Persona research and user research — owned by a separate research phase
- CLI information architecture — owned by a future `CLI_IA.md` skill
- Native and mobile-app information architecture — this skill is web-only (routes, URLs, browser, responsive)
- Database schema, business logic, algorithm design — owned by architecture and unit SPECs
- Deployment and routing infrastructure (CDN, edge workers) — owned by DEPLOYMENT docs

---

## Quality Checklist

- [ ] Output file has valid YAML frontmatter with all required fields (`skill`, `date`, `status`, `pages_documented`, `flows_documented`, `shared_surfaces`, `use_cases_covered`, `use_cases_total`, `auth_pages`, `public_pages`, `open_questions`)
- [ ] No placeholders, TODOs, or vague language ("appropriate", "relevant", "as needed", "etc.")
- [ ] Open Questions section is present (empty or with genuine ambiguities only)
- [ ] Output is self-contained — readable and actionable without opening other files
- [ ] Every page template in the sitemap table has a full per-page blueprint entry below
- [ ] Every per-page entry declares route pattern, URL parameters, auth requirement, primary CTA (or explicit chrome/nav/informational classification), sections, data dependencies, and the full state matrix from Rule 3
- [ ] Every page with a dynamic route documents parameter-dependent behavior (at minimum: not-found and unauthorized branches)
- [ ] Every use case in the input use cases document maps to at least one page in the traceability matrix, or is explicitly listed as having no web surface with a one-line reason
- [ ] Every page in the inventory is referenced by at least one use case in the traceability matrix, or is explicitly classified (chrome, auth, error, informational)
- [ ] URL strategy is stated once and every route in the sitemap conforms to it (or carries an explicit exception note)
- [ ] Global navigation is documented in exactly one section; per-page entries reference it by name and do not re-describe it
- [ ] Every flow step references an existing page template from the inventory (no flow introduces new pages)
- [ ] Shared UI surfaces appear only in the dedicated Shared UI Surfaces section, not scattered in per-page entries
- [ ] Conditional sections (accessibility, SEO, i18n, telemetry, authenticated context) are either fully completed or fully omitted — never present-and-empty
- [ ] No component-level details (props, hooks, component APIs) appear anywhere in the document
- [ ] No visual-design decisions (colors, font families, spacing values, component library names) appear anywhere in the document
- [ ] No final UX copy — section and action roles are described, not literal button labels or headings
- [ ] Frontmatter `use_cases_covered` and `use_cases_total` are present; any gap is explained in the Traceability Matrix
