# Enterprise React Architecture Guide

*A stack-flexible, principle-driven architecture reference for building React
applications that stay scalable, maintainable, and fast as an enterprise
codebase, its team, and its business domain all grow.*

This guide is written to be dropped into any enterprise React project,
regardless of industry or backend. The concrete libraries named in
[Recommended Technology Stack](#recommended-technology-stack) are a strong,
justified default — not a hard requirement. Swap any of them for an
equivalent that better fits an existing team's constraints, as long as the
underlying principle behind the choice survives the swap. Nothing else in
this guide depends on any single library.

## How to Use This Guide

- Adopt as much or as little as fits your project. [What Must Be
  Centralized](#what-must-be-centralized) and [Architectural
  Rules](#architectural-rules) carry the most weight — every other section
  supports one of those two.
- This guide is written entirely in **English**, and every example
  identifier in it is English, so it is ready to use as-is on any project.
- The one exception is **[Appendix A](#appendix-a--french-language-naming-convention-french-client-projects-only)**,
  which documents a French-language naming convention for code identifiers.
  It applies **only** to projects delivered to French-speaking end users
  where the team has also chosen to write business-domain *code* in French,
  not just user-facing text. **If that isn't this project, delete Appendix A
  outright** — nothing else in this guide references it or depends on it.
- Everything is a starting point, not scripture. Where this guide's default
  disagrees with a constraint your project actually has, write down the
  deviation (see [Documentation Governance](#documentation-governance--definition-of-done))
  and move on.

---

## Contents

1. [Core Architectural Goals](#core-architectural-goals)
2. [What Must Be Centralized](#what-must-be-centralized)
3. [Recommended Technology Stack](#recommended-technology-stack)
4. [Project Structure](#project-structure)
5. [Module & Feature Convention](#module--feature-convention)
6. [Scaling Model — From One Module to an Enterprise System](#scaling-model--from-one-module-to-an-enterprise-system)
7. [Cross-Module Communication](#cross-module-communication)
8. [Route Grouping](#route-grouping)
9. [Layer Responsibilities](#layer-responsibilities)
10. [Architectural Rules](#architectural-rules)
11. [Startup & Configuration Safety](#startup--configuration-safety)
12. [HTTP & Data Fetching](#http--data-fetching)
13. [Data Layer at Scale](#data-layer-at-scale)
14. [Forms & Tables at Enterprise Scale](#forms--tables-at-enterprise-scale)
15. [Error-Handling Taxonomy](#error-handling-taxonomy)
16. [Permissions & Conditional UI](#permissions--conditional-ui)
17. [Routing](#routing)
18. [Centralized Route Paths](#centralized-route-paths)
19. [Multi-Tenancy & Active Scope (Optional)](#multi-tenancy--active-scope-optional)
20. [Entity Workflows & State Machines (Optional)](#entity-workflows--state-machines-optional)
21. [Design Tokens & Theming](#design-tokens--theming)
22. [Backend Integration Contract](#backend-integration-contract)
23. [Performance Budgets](#performance-budgets)
24. [Naming Conventions & Code Style](#naming-conventions--code-style)
25. [Testing Organization](#testing-organization)
26. [Tooling & Automation](#tooling--automation)
27. [Documentation Governance & Definition of Done](#documentation-governance--definition-of-done)
28. [Dead Code Hygiene](#dead-code-hygiene)
29. [Architecture Self-Check](#architecture-self-check)
30. [Appendix A — French-Language Naming Convention (French Client Projects Only)](#appendix-a--french-language-naming-convention-french-client-projects-only)

---

## Core Architectural Goals

Every recommendation in this guide exists to serve one of these four goals.
When two pieces of advice seem to conflict, resolve the conflict in favor of
whichever goal matters more for the change at hand.

- **Scalability** — new modules, features, and teams add code without
  adding coupling. Doubling the number of business domains should not
  double the effort of understanding any single one of them.
- **Maintainability** — a change in one place has a small, predictable
  blast radius, and deleting a feature is safe (see [Architectural
  Rules](#architectural-rules), Rule 9).
- **Performance** — the app stays fast as it grows, by enforced budget, not
  by luck (see [Performance Budgets](#performance-budgets)).
- **Safety** — a bad request, an invalid config value, or a bug in one
  corner of the app degrades gracefully instead of producing a blank screen
  or corrupted data (see [Startup & Configuration
  Safety](#startup--configuration-safety) and [Error-Handling
  Taxonomy](#error-handling-taxonomy)).

The point of folder structure, in particular, is never organization for its
own sake — it's **clear ownership**, **strict boundaries**, **low
coupling**, **easier refactoring**, and **safe deletion**.

---

## What Must Be Centralized

The single biggest predictor of whether an enterprise frontend stays
maintainable at scale is whether the concerns below live in **exactly one
place**. Duplicate any of them across features and the codebase accumulates
silent inconsistency long before it accumulates visible bugs — two slightly
different date formatters, three ways of reading an environment variable,
a permission check that only half the pages remembered to add.

| Concern | Centralize in | Why one place |
| --- | --- | --- |
| Environment & configuration validation | One config module | So an invalid deploy fails predictably in one spot instead of surfacing as a mystery bug deep in a feature. |
| HTTP transport & response parsing | One base API client | So the backend's response shape (envelope, error format) is understood in exactly one layer — see [HTTP & Data Fetching](#http--data-fetching). |
| Authentication/session state & token lifecycle | One session module | Token refresh races and logout edge cases are hard enough to get right once. |
| Permission *evaluation* engine | One core module | The engine stays a pure, business-blind string comparator; the *vocabulary* of what each permission means stays distributed per module — see [Permissions & Conditional UI](#permissions--conditional-ui). |
| Route path strings | One path registry | Renaming or nesting a route becomes a one-line change instead of a project-wide search. |
| Route tree assembly | One route configuration, fed by per-module fragments | One place shows the whole app's navigable surface at a glance. |
| Design tokens / visual theme | One token file + a generated theme | A rebrand, a dark mode, or a UI-library swap touches one file instead of hundreds — see [Design Tokens & Theming](#design-tokens--theming). |
| Error-handling taxonomy | One mapping of error category → UI treatment | So "how do we show this failure" is a lookup, not a fresh decision per feature — see [Error-Handling Taxonomy](#error-handling-taxonomy). |
| Logging / observability sink | One logging abstraction | So redaction, formatting, and the eventual choice of logging backend are decided once. |
| Active tenant/business scope *(if multi-tenant)* | One scope provider | Prevents one tenant's cached data from ever leaking into another tenant's view — see [Multi-Tenancy & Active Scope](#multi-tenancy--active-scope-optional). |
| Query-key factory pattern | One factory per module, following one shared shape | Keeps cache invalidation predictable once the app has hundreds of queries — see [Data Layer at Scale](#data-layer-at-scale). |
| Generic, truly shared i18n copy | One shared i18n layer | Ownership of most copy still stays distributed (feature → module → shared) — see Rule 17 — but the *generic* tier (a handful of universal words) belongs in exactly one file. |
| Feature-flag evaluation *(if used)* | One provider | So a flag's value can never disagree with itself between two parts of the same page. |

Everything **not** on this list is deliberately *not* centralized: business
logic, permission vocabulary, feature-local copy, and feature-local
validation schemas are meant to live next to the code that owns them (see
[Module & Feature Convention](#module--feature-convention)). Centralizing
those too would recreate the tightly coupled "god folder" this architecture
is built to avoid.

---

## Recommended Technology Stack

Every row below is a default with a reason, not a mandate. Swap freely —
the rest of this guide is written around the *pattern*, not the *library*.

| Category | Recommended default | Why | Reasonable alternatives |
| --- | --- | --- | --- |
| Framework / rendering | React (current stable), client-rendered single-page app via a modern build tool | Most enterprise apps sit behind auth, so SEO and pre-JS first paint rarely matter — a plain SPA keeps the mental model and deploy story simple. | If part of the app is public-facing and needs SEO or a fast first paint (a marketing site, a public status page), use a server-rendered React framework for just those routes, or the whole app, while keeping every other principle in this guide unchanged. |
| Language | TypeScript, strict mode on | The layering and boundary rules in this guide lean on the compiler to catch accidental coupling. | Not seriously recommended to skip at this scale. |
| UI component library / design system | Pick one and commit | As long as [Design Tokens & Theming](#design-tokens--theming) is followed, swapping the library later only touches the theming layer, since features never hardcode raw visual values. | Any mature library (a Material-style kit, an Ant Design-style kit, Chakra, Mantine) or a utility-CSS + headless-component setup all work equally well under this architecture. |
| Server-state / data fetching | TanStack Query (React Query) | The de facto standard for a reason: caching, deduplication, retries, and cancellation are hard to get right by hand, and reinventing them per feature is a common source of enterprise bugs. | A lighter library with a smaller API surface; an RTK-Query-based approach if the team is already committed to a Redux-based store for other reasons. |
| Low-level HTTP | A thin, envelope-aware wrapper around the platform's native fetch | A data-fetching library already owns caching/retry/cancellation, so a second HTTP library with its own interceptor system is usually redundant weight. | A promise-based HTTP client is a fine swap when a real need exists — upload-progress events, an existing large interceptor chain, or very old browser support. The principle that matters is *one* low-level HTTP layer, not which library implements it. |
| Client-side validation | A schema-validation library whose inferred types double as your form/DTO types | Keeps a form's runtime validation and its TypeScript types from ever drifting apart. | A smaller, less mature alternative is worth it only if validation-library size actually shows up in your performance budget. |
| Global client state | Nothing, by default | Server state plus component-local state covers most enterprise screens; a global store added "just in case" is unused complexity. | When a genuine cross-cutting UI need appears (a multi-step flow spanning routes, a persisted preference), reach for the lightest store your team is comfortable debugging — a minimal atom/store library is a reasonable default; a stricter, more ceremonious store remains reasonable for teams that specifically want its conventions and devtools at very large scale. |
| Routing | A React SPA router, wrapped in the centralized declarative route pattern in [Routing](#routing) | The centralization pattern is what actually matters, and it works with any router that supports a data-driven route tree. | — |
| Forms | The UI library's own form primitives, or a lightweight form-state library, both driven by the same schema used for validation | Avoids a second, parallel schema language for forms. | — |
| Testing | A Vite-native unit/component test runner + a user-behavior-focused component testing library for units; a modern cross-browser tool for end-to-end smoke tests | Shares config and speed characteristics with a Vite-based build; keeps tests focused on user-observable behavior instead of implementation detail; strong current cross-browser e2e coverage. | Any well-maintained equivalent in the same category. |
| Linting / formatting | A linter capable of **custom, path-based rules**, paired with a formatter | The cross-feature-import ban, the shared-layer dependency direction, and the DTO-boundary rule (see [Architectural Rules](#architectural-rules)) only hold long-term if a machine enforces them on every change — code review alone drifts. As of this writing, the most mature custom-rule ecosystem is still the traditional JS/TS linting tooling; a combined, faster toolchain is worth adopting once its custom-rule support covers your specific boundary rules. | — |
| Git hooks & commit convention | A pre-commit hook that runs lint/format only on staged files, plus a commit-message hook enforcing a consistent convention (Conventional Commits is a reasonable default) | Keeps commit history searchable and CI fast. | — |
| CI | Whatever your organization already standardizes on | Running lint, typecheck, test-with-coverage, and build on every push/PR is the requirement — the specific provider is not architecturally significant. | GitHub Actions, GitLab CI, Azure DevOps, Bitbucket Pipelines, and so on are all fine. |

---

## Project Structure

- `.env`, `.env.example` — actual configuration values and a kept-in-sync
  example file; only non-sensitive example values are ever committed. Both
  are read exclusively through the config-validation module (see [Startup &
  Configuration Safety](#startup--configuration-safety)) — nothing else in
  the codebase reads the raw environment directly.
- `src/`
  - `app/` — providers, app composition, the top-level error boundary, the
    mounted theme and data-fetching-client instances.
  - `bootstrap/` — startup logic that runs once the environment is
    confirmed valid, before the app renders (e.g. startup logging setup).
  - `core/` — the technical, business-agnostic foundation: configuration
    validation and the startup failure screen; a logging abstraction; the
    permission engine and its thin React bindings; the real
    authentication/session module (token lifecycle, refresh handling); an
    active-scope provider, only if the app is multi-tenant (see
    [Multi-Tenancy & Active Scope](#multi-tenancy--active-scope-optional)).
  - `api/base/` — the single low-level HTTP client: request methods,
    timeout handling, header-provider injection, an idempotency-key
    opt-in, a typed error parser, a params serializer.
  - `shared/contracts/` — protocol-level types shared by `api/base` and any
    shared UI that renders server data generically (a response-envelope
    type, pagination shapes, a generic filter-definition type).
  - `design-system/` (or your UI library's equivalent) — the design-token
    file and the generated theme, plus any small wrapper components the
    whole app needs.
  - `layouts/` — app shells (a main authenticated layout with its
    navigation, an auth-flow layout) — pure wrappers with no data fetching
    of their own.
  - `shared/` — purely technical, globally reusable, business-agnostic
    building blocks: generic UI primitives, generic data-heavy composables
    if your domain needs them (a server-driven data table, a
    metadata-driven filter panel, string-preserving amount/quantity
    inputs, four-state view wrappers, a status badge, a workflow action
    bar, an entity picker), generic hooks, formatters, and the shared
    i18n layer.
  - `pages/` — top-level pages wired through the route configuration that
    don't yet belong to a specific business module.
  - `modules/` — the business domain layer; see [Module & Feature
    Convention](#module--feature-convention).
  - `routes/` — the path registry, the central route-configuration
    assembler, the router wiring, and route guards.
  - `styles/` — any global, non-component styling.
  - `test-support/` — shared testing infrastructure: a fetch-mocking
    router and its handlers, shared fixtures.
  - the framework's entry file.

---

## Module & Feature Convention

Business code lives under `modules/`. Each subfolder is a fully isolated
**bounded context** (e.g. `customers`, `billing`, `scheduling`). Inside a
module, split code by concern:

- `modules/<context>/`
  - `api/` — hand-written DTO types matching the backend response exactly,
    endpoint functions, mapper functions (DTO → domain entity), and this
    module's query-key factory (scope-prefixed if the app is
    multi-tenant).
  - `i18n/` — copy shared across this module's features.
  - `domain/` — pure business logic: entities, small domain helper
    functions, and this module's permission-key vocabulary — completely
    framework-free.
  - `shared/` — code reused across this module's features but not meant to
    leave the module.
  - `features/<feature>/` — one folder per user-facing capability within
    the module; see the feature-level rows of [Layer
    Responsibilities](#layer-responsibilities).
  - `pages/` — page components that compose this module's features and
    evaluate page-level permissions.
  - `routes.tsx` (or equivalent) — this module's route fragment, carrying
    navigation metadata, mounted by the central route configuration.
  - `index.ts` — the module's gateway: only the symbols other modules are
    actually meant to consume.

A feature, in turn, is a self-contained slice inside a module: a list view,
a creation flow, a detail editor. Two features in the same module never
import from each other directly (see Rule 1) — anything they both need
bubbles up to `module/shared/` or `module/domain/` (Rule 2).

---

## Scaling Model — From One Module to an Enterprise System

The same folder shape scales without being restructured:

- **One module.** A single bounded context, built exactly as described
  above. Nothing here is "the small version" of a larger pattern — it's the
  same pattern.
- **Several modules, one team.** Cross-module rules start to matter, since
  two modules will inevitably need each other's data or navigation — see
  [Cross-Module Communication](#cross-module-communication).
- **Many modules, many teams.** The route tree needs splitting (see [Route
  Tree at Enterprise Scale](#routing)), and ownership needs to be explicit
  — see [Documentation Governance](#documentation-governance--definition-of-done).

The point of this layering is that none of the earlier stages need to be
revisited to reach the next one: modules are **added**, not the foundation
re-architected.

---

## Cross-Module Communication

Pick a rough layering of your own domain — for example, transactional
modules (orders, payments) may depend on reference modules (customers,
catalog, settings), never the reverse — and enforce that direction. A
circular dependency between two modules is almost always a sign that the
shared piece belongs in `shared/`, or in a new, lower-level module both can
depend on.

When one module genuinely needs something from another, use one of these
sanctioned mechanisms only:

1. **Gateway import** — import only through the other module's `index.ts`,
   never its internals.
2. **URL navigation** — link to a page in the other module instead of
   importing its component tree directly.
3. **Query-cache invalidation** — invalidate a shared, well-known query key
   across the boundary instead of calling a function directly.
4. **Promotion to `shared/`** — once a piece of code is genuinely needed by
   more than one module (not just anticipated to be), promote it to the
   global `shared/` layer, applying the same "bubble up on the second real
   consumer" reasoning as Rule 2, one level higher.

---

## Route Grouping

Use parenthesized folder names — `(dashboard)`, `(auth)`, `(marketing)` —
purely to organize the module/page folder layout conceptually, without those
names leaking into the actual URL. Most enterprise SPAs declare routes
explicitly in a central configuration rather than inferring them from the
file tree, so a group folder here is an organizational convention for
*code layout*, not a routing mechanism — the central route configuration
still owns the literal path tree.

Why bother:

- Keeps related business areas together without leaking that grouping into
  URLs.
- Separates dashboard, public, and auth concerns cleanly as the app grows.
- Makes ownership clearer for teams working on different domains.
- Lets a non-business grouping (a folder of shared example or reference
  content) be marked just as clearly as a real layout concern.

---

## Layer Responsibilities

The examples below use a generic `customers` module for illustration.

### Root & Infrastructure Layers

| Folder / Layer | Responsibility | Contains | Forbidden |
| --- | --- | --- | --- |
| `app/` | Application composition layer. | Providers, app composition, the error boundary, the mounted theme-provider and data-fetching-client instances. | Business logic, feature workflows, domain rules, feature state. |
| `bootstrap/` | Startup logic run once the environment is confirmed valid. | Startup logging registration and similar one-time setup. | UI components, layouts, hooks, anything assuming an invalid environment might still reach it. |
| `core/` | Technical, business-agnostic system foundation. | Config validation + startup failure screen; logging abstraction; the permission engine and its thin React bindings; the real session module (token lifecycle, refresh handling); an active-scope provider if multi-tenant. | Styled/visual UI components, business-specific models, feature execution. |
| `api/base/` | Low-level network infrastructure — the only layer that parses the backend's response shape. | The base HTTP client (GET/POST/PUT/PATCH/DELETE, timeout, header-provider injection, idempotency-key opt-in), a typed error hierarchy, a binary-download helper, a params serializer. | Business rules, entity types, JSX, a second overlapping HTTP library (see [HTTP & Data Fetching](#http--data-fetching)). |
| `shared/contracts/` | Neutral protocol types shared by `api/base` and generically server-data-aware shared UI. | A response-envelope type, page/cursor response shapes, a generic filter-definition type. | Endpoint functions, runtime logic. |
| `design-system/` | Application theming layer and design tokens. | The single token file, the generated theme (built only from tokens plus documented overrides), a thin wrapper component. | Business logic, domain rules, feature state, raw hex/px values in features. |
| `layouts/` | App shells. | A main authenticated layout (navigation driven by route metadata, a topbar), an auth-flow layout. | Data fetching beyond shell needs, route declarations, business rules. |
| `shared/` | Purely technical, globally reusable, business-agnostic items. | Generic UI primitives; generic data-heavy composables (a server-driven table, a metadata-driven filter panel, string-preserving amount/quantity inputs, four-state wrappers, a status badge, a workflow action bar, an entity picker); generic hooks; formatters; the shared i18n layer. May type-only import from `shared/contracts`. | Domain-aware items, business constraints, and anything from `core/`, `modules/`, `app/`, or `routes/` (lint-enforced) — runtime imports from `api/base` are forbidden too, for the same dependency-inversion reason. |
| `pages/` | Top-level pages wired through the route configuration. | Landing/home pages and any page not yet belonging to a feature module. | Business domain logic — prefer `modules/` for that. |
| `modules/` | Business domain layer. | Each subfolder, following the Module & Feature Convention, including its own data access. Gateways export only consumed symbols; cross-module dependency direction is downstream only. | Cross-module direct file imports bypassing a module's gateway; upstream dependencies from reference modules onto transactional modules. |
| `routes/` | Centralized route declarations. | The route-configuration assembler, the path registry, the router wiring, route guards. | Hardcoded path strings anywhere else in the app, business logic. |
| `module/i18n/` | Copy shared across a module's features. | Shared page titles, shared labels. | Feature-exclusive strings. |
| `module/pages/` | Page composition; orchestrates features and evaluates page-level permissions. | A list page composing a list feature and a create feature, gating both with the permissions hook/guard. | Directly handling API calls, complex data orchestration. |
| `module/domain/` | Pure, framework-independent business core, including this module's permission vocabulary. | The domain entity, small domain helper functions, this module's permission-key constants. | React components, state managers, API requests. |
| `module/shared/` | Reusable code constrained to this module's features. | A shared status-tag component, a shared option list. Query-key factories live in `api/`, not here. | Infrastructure wrappers, globally generic components. |

### Feature-Level Layers (`modules/*/features/*`)

| Folder / Layer | Responsibility | Contains | Forbidden |
| --- | --- | --- | --- |
| `feature/domain/` | Feature-local domain rules. | A search-filtering function local to this one feature. | React context, global actions. |
| `feature/i18n/` | Micro-copy exclusive to this feature. | Button labels, inline messages, error/retry copy. | Shared global/module strings. |
| `feature/components/` | Granular, dumb UI blocks. | A card, a form — permission-driven visibility comes in as a plain boolean prop, never a hook call inside the component. | Network logic, global state sync, permission-hook calls (Rule 11). |
| `feature/sections/` | Larger UI structures assembling components, including distinct loading/error/empty states. | A table section, a creation-modal section. | Complex domain logic. |
| `feature/application/` | Workflow coordinator: orchestrates the data-fetching library, modal state, form submission. | A list query hook (with an option to skip fetching when unauthorized), a creation mutation hook (optimistic only where Rule 24 allows), a workflow hook combining modal + form + submit. | Raw HTML/primitive rendering. |
| `feature/state/` | Feature-specific client state, when genuinely needed. | Local derived UI state managers. | Duplicating the data-fetching library's server cache. |
| `feature/hooks/` | Hooks scoped to this one feature. | A feature-local filter hook. | Globally generic utilities. |
| `feature/utils/` | Stateless helpers. | A function bridging a validation schema into the form layer's validator shape. | State management, browser mutations. |
| `feature/validation/` | Runtime input validation. | This feature's validation schema. | Cross-feature shared schemas. |
| `feature/types/` | Types local to this feature. | Component props types. | Sharing outside the feature without promoting to `module/`. |
| `feature/constants/` | Feature-local constants. | Initial form values, static option lists. | Global system parameters. |
| `feature/tests/` | Tests for this feature. | Unit tests, component tests. | Cross-module test setups. |
| `feature/index.ts` | Public boundary. | Explicit named exports. | Blind wildcard re-exports. |

---

## Architectural Rules

- **Rule 1 (No Reverse / Cross-Feature Imports):** Features never import
  from sibling features directly. Enforce it with a lint rule so a
  violation fails CI, not just code review — add a new rule entry whenever
  a feature must not reach into a sibling.
- **Rule 2 (Feature Isolation via Bubbling):** Share code between features
  only by promoting it to `module/shared/` or `module/domain/`, and only
  once a second real consumer actually exists — not preemptively.
- **Rule 3 (Strict Downward Dependency Flow):** The global `shared/` layer
  stays completely business-agnostic and never imports from `modules/`,
  `app/`, or `routes/` — nor from the parts of `core/` that carry
  session/business awareness (which is exactly why the permission guard
  component lives in `core/permissions/`, not `shared/`, even though it
  renders no visual markup of its own). Lint-enforced.
- **Rule 4 (Domain Purity):** Anything under a `domain/` folder — module- or
  feature-level — stays 100% framework-free: no React, no hooks, no query
  client, no direct API calls. It should be testable in plain TypeScript
  with zero DOM.
- **Rule 5 (Thin View Layers):** Pages and top-level components compose and
  orchestrate; they don't embed business rules, data mapping, or
  validation logic directly in JSX.
- **Rule 6 (Pragmatic State Placement):** Choose where state lives by its
  lifecycle:

  | State Type | Recommended Tool |
  | --- | --- |
  | Local UI state | Component state (`useState`/`useReducer`) |
  | Server cache | The data-fetching library |
  | Global app state | A dedicated store, added only when justified |
  | Complex feature state | `feature/state/` |

- **Rule 7 (Server State Source of Truth):** The data-fetching library owns
  all server-derived data. Never copy a query result into a second state
  container "for convenience" — write directly into the query cache for an
  optimistic update instead.
- **Rule 8 (Controlled Public Surfaces):** Every module and feature exposes
  an explicit gateway file listing exactly what it re-exports. Blind
  wildcard re-exports are banned.
- **Rule 9 (Safe Deletability):** Deleting a feature folder must never
  break a sibling feature; deleting a whole module must never break
  anything outside it. Treat this as the acceptance test for Rules 1, 2,
  and 8 together.
- **Rule 10 (Encapsulate Declarative Checks):** Don't scatter raw
  role/permission comparisons through JSX. Route every conditional-render
  decision through the permission engine's helpers (see [Permissions &
  Conditional UI](#permissions--conditional-ui)).
- **Rule 11 (Presentation-First UI):** Leaf, purely presentational
  components take booleans and callbacks as props; they never call a
  permission hook, a data hook, or reach into global state themselves.
  Evaluation happens once, at the page/section orchestration boundary, and
  flows down as props.
- **Rule 12 (Module Gateways):** Every module exposes one explicit entry
  file. External layers only ever import through it.
- **Rule 13 (No Architectural Dumping Grounds):** Top-level folders named
  `misc`, `helpers`, `common`, `other`, or `utils` are banned — also
  caught by a lint rule on the import path.
- **Rule 14 (DTO Sandbox Boundary):** Raw backend response shapes never
  reach UI components directly — each module owns its own DTO types and
  its own mapper functions. Importing another module's DTOs or endpoint
  functions directly is forbidden; go through that module's gateway.
- **Rule 15 (Lightweight Layouts):** A reusable page shell is a pure
  wrapper with no data fetching of its own, attached to a group of routes
  once in the central route configuration — never re-wrapped by every page
  that uses it.
- **Rule 16 (Business-Blind Infrastructure):** `core/` and `api/base/`
  never reference a specific domain concept. The permission engine compares
  opaque strings with zero idea what any given string means — only the
  owning business module assigns meaning to its own permission keys. This
  is what keeps `core/` reusable across an unrelated project without
  modification.
- **Rule 17 (Distributed Localization Ownership):** Feature-local copy
  lives in the feature; copy shared across a module's features lives at
  the module level; truly generic words live in the shared i18n layer.
  Keep the shared layer minimal — a generic label with no current consumer
  is dead code (see [Dead Code Hygiene](#dead-code-hygiene)), not a head
  start. All tiers hold whatever language the application actually ships
  to end users — see [Appendix A](#appendix-a--french-language-naming-convention-french-client-projects-only)
  if that's French and the team has also chosen to name business-domain
  code in French.
- **Rule 18 (Permission-Driven UI Composition):** Evaluate permissions
  once, at the orchestration boundary, then pass the result down as props
  or wrap JSX with a guard component — never call the permissions hook
  from inside a purely presentational leaf.
- **Rule 19 (Centralized Route Paths):** Never hardcode a path string in a
  route declaration, a navigation call, or a link outside of the single
  path registry. A shared not-found/access-denied component linking to the
  literal root path is a documented, deliberate exception, since it must
  not depend on `routes/` (Rule 3).
- **Rule 20 (Response Envelope Boundary):** Whatever shape your backend's
  responses take, parse it in exactly one place — the base API client.
  Every other layer consumes already-unwrapped data and a typed error,
  never a raw response body.
- **Rule 21 (Scoped Context Headers & Keys — if multi-tenant):** Any
  tenant/scope header is injected exclusively by the base API client via a
  centrally registered provider. Features never attach scope headers
  manually, and every query key embeds the active scope through the
  module's key factory — switching tenants can never leak one tenant's
  cached data into another's view.
- **Rule 22 (Idempotent Critical Creations):** Any creation endpoint for a
  financial or otherwise hard-to-undo resource (an order, an invoice, a
  payment) carries a client-generated idempotency key, stable across
  retries of the same submission attempt. A detected replay surfaces as an
  informational notice, never a silent duplicate.
- **Rule 23 (Precise Numeric & Money Handling):** Monetary and other
  precision-sensitive values are treated as exact strings end-to-end —
  never run through floating-point math client-side, and never compute a
  total the server didn't already provide.
- **Rule 24 (Restricted Optimistic Updates):** Optimistic UI updates are
  reserved for low-risk, easily reversible mutations. Anything touching
  inventory, money, or another system of record waits for the server's
  response, then invalidates the narrowest correct cache keys.
- **Rule 25 (Permission Keys Verbatim From Backend):** Client-side
  permission keys are the backend's own catalog, imported verbatim from
  the owning module's constants — never invented independently on the
  frontend. The server re-checks every permission on every request; hiding
  a control client-side is UX, never a security boundary on its own.
- **Rule 26 (Design Tokens as Single Source of Truth):** Colors, spacing,
  control heights, radii, shadows, and typography come from the
  design-token layer — never hardcoded as raw values in feature code.
  Status-to-color mappings live in the owning module, mapped onto generic
  semantic tokens, never onto raw colors.

---

## Startup & Configuration Safety

A common trap: validating environment variables by throwing a plain error
at module-import time. That failure happens *before* the framework's root
render call even runs, so the app's own error boundary never gets a chance
to catch it — the result is a fully blank screen with nothing on-screen,
only a stack trace in the devtools console.

The fix:

- The environment-validation function is pure, returns a discriminated
  result (success-with-values, or failure-with-errors), and **never
  throws**. Being decoupled from the raw environment also makes it directly
  unit-testable with arbitrary fixtures.
- The computed result is produced once, still without throwing.
- The app's single entry point is the one place that checks that result:
  if it failed, it renders a dependency-free error screen — deliberately
  free of any other app dependency (no theme, no design tokens, no i18n)
  so it can never itself fail to render — instead of mounting the app.

**Rule of thumb:** nothing in `core/` (or anywhere else) should throw
synchronously at module scope. If a piece of startup configuration can be
invalid, model it as data and let one deliberate place at the top of the
tree decide what to render.

---

## HTTP & Data Fetching

Keep exactly one low-level HTTP layer — a small, envelope-aware wrapper
around a request primitive (native fetch by default; see [Recommended
Technology Stack](#recommended-technology-stack) for when a full HTTP
library is a better fit). It unwraps the response envelope once, injects
cross-cutting headers via centrally registered providers, and throws a
typed error hierarchy rather than letting features branch on a raw response
body.

**Why keep it thin:** a data-fetching library like TanStack Query already
owns caching, deduplication, retries, and cancellation on unmount — the
exact surface area a full-featured HTTP library's interceptor system is
often reached for. Stacking both means paying for two overlapping systems
for no real benefit.

**The pattern for any new endpoint**, described without code:

1. A protocol-only function in the module's `api/` layer calls the base
   client and returns the raw DTO shape — no React, no business logic.
2. A mapper function in the same layer converts that DTO into the module's
   domain entity, and normalizes any backend error shape into one
   consistent structure.
3. A query or mutation hook in the feature's `application/` layer wraps
   step 1–2 in the data-fetching library, owning its own cache key (from
   the module's key factory) and its staleness tier.

**Error states are not optional.** Any list/detail view must branch on
loading → error → empty → data as four distinct states, with a retry action
wired to the data-fetching library's refetch. Collapsing "the request
failed" into "there's no data" is a real bug — a silently empty page on a
failed request is exactly the kind of defect this architecture exists to
prevent.

---

## Data Layer at Scale

Conventions that keep hundreds of queries coherent as modules accumulate.

### Query keys: one factory per module

Keys are structured, never ad hoc arrays. Each module owns one key factory,
nested hierarchically so coarse invalidation stays safe: an unscoped module
root, a "list" branch parameterized by the current filters, and a "detail"
branch parameterized by the entity's id. Invalidating the detail branch as a
whole hits every detail view without touching any list.

If the app is multi-tenant, every factory prefixes its keys with the active
scope (see [Multi-Tenancy & Active Scope](#multi-tenancy--active-scope-optional)
and Rule 21) — combined with the scope-switch semantics described there,
this makes cross-tenant cache bleed impossible by construction.

### Staleness tiers

| Data kind | Example | Staleness window | Rationale |
| --- | --- | --- | --- |
| Reference / config | payment terms, tax rates, user preferences | 5–15 min | Rarely changes mid-session; refetching on every mount is waste. |
| Transactional lists | invoices, orders, stock | 30–60 s | Must feel live but tolerate short staleness; provide an explicit refetch control anyway. |
| Detail being actively edited | an open record | Always fresh, plus targeted invalidation | Editing demands freshness; mutations invalidate precisely rather than relying on a timer. |

### Pagination & big lists

Server-side pagination is the default for anything that can exceed a few
hundred rows, with the total count surfaced by the backend and page size
clamped to its maximum. Cursor-based infinite loading is reserved for
genuinely append-only, ordered feeds (an activity log, a running ledger),
triggered by an explicit "load more" action — never auto-infinite-scroll —
and capped at a maximum page count. Infinite scroll is otherwise avoided
for dense enterprise grids, where predictable scan position matters more
than a seamless feed.

### Invalidation etiquette

Mutations invalidate the **narrowest** correct key set (a detail key, not
the whole module root), and do it on success — not fire-and-forget.
Optimistic updates are restricted per Rule 24.

---

## Forms & Tables at Enterprise Scale

Enterprise apps live in big forms and dense grids. The sanctioned patterns:

### Forms

- One validation schema per form, composed from module-shared base schemas
  when entities overlap (a billing address appearing in three creation
  flows gets one shared schema, not three copies).
- Long forms (roughly 8+ fields, or more than one logical group) render as
  a stepper or tabs of section components, each validating independently
  against a subset of the same schema — never duplicate field definitions
  across sections.
- Unsaved-changes guards are promoted to a shared hook only once a second
  consumer exists; until then they live in the first feature that needs
  them (see [Dead Code Hygiene](#dead-code-hygiene)).
- Submission is a feature-level mutation hook; the form component receives
  its submitting state and server-side errors as props (Rules 5, 11).
  Server-side field errors map onto form fields through the mapper layer,
  never by parsing raw responses inside the component — the mapper
  normalizes every backend error shape into one consistent structure.
- Money and quantity inputs preserve their exact string value rather than
  parsing to a float on every keystroke (Rule 23). Destructive or terminal
  actions go through a confirmation dialog, with a required justification
  input when the domain calls for an override.
- Creations on idempotent resources attach their key via a small shared
  hook (Rule 22) — never hand-managed per feature.

### Tables (grids)

- **Filters ideally come from backend-provided metadata**: a listable
  resource exposes what fields it can be filtered on, and the UI renders
  controls from that metadata instead of hardcoding filter enums that can
  silently drift out of sync with the backend.
- Server-side filtering, sorting, and pagination for any entity that can
  pass a few hundred rows. Filter/sort/page state lives in the URL's search
  params — shareable, back-button-safe views.
- Row selection is feature state, never cached as if it were server data
  (Rule 7).
- Virtualize cell rendering past roughly 20 visible columns or 100
  unpaginated rows. Add a virtualization dependency only when a real grid
  needs it.
- Every grid implements the same four states as list views: loading, error
  with retry, empty, and data.

---

## Error-Handling Taxonomy

Every failure falls into exactly one bucket, with one sanctioned handler:

| Kind | Detected where | Rendered how | Never |
| --- | --- | --- | --- |
| Network / timeout | Base API client | Section-level error state + retry | A toast that vanishes while the page looks empty |
| Backend validation (4xx payload) | Mutation error handler | Field-level errors mapped onto the form | A generic "something went wrong" for a recoverable case |
| Permission denied | Permissions hook / route guards | An access-denied page, or a hidden control | Showing disabled controls the user can never enable |
| Unexpected (bug, 5xx, thrown render error) | Error boundary / data-fetching library defaults | Boundary fallback with a report action; logged | Swallowing into the console only |
| Environment / config | Startup validation | The dependency-free startup error screen | Anything else rendering first |

**Rule of thumb:** errors a user can act on render inline where the action
is; errors they can't act on escalate to a boundary. Toasts are reserved
for success confirmations and background-action failures (an auto-save),
never for primary view loading failures.

---

## Permissions & Conditional UI

Three tiers, each staying inside its own layer's rules:

1. **The engine** — pure functions comparing a permission string against a
   list of permission strings. Zero business knowledge: it has no idea
   what any given string means.
2. **The React bindings** — a hook plus a guard component. The hook reads
   the current session's permission list and exposes the engine's
   functions bound to it; the guard component wraps the common case of
   conditionally rendering a JSX subtree.
3. **The vocabulary** — each module's own permission-key constants. Only
   the owning module knows what a given key actually means (Rule 25). This
   is what keeps the engine reusable across any project without
   modification (Rule 16).

**When to use which:**

| Situation | Use |
| --- | --- |
| Toggling a chunk of JSX (a button, a section) | The guard component |
| Need the boolean itself — to gate a query, combine with other conditions, or pass down as a prop | The permissions hook directly |
| A leaf, purely presentational component needs to vary by permission | A plain boolean prop from the page/section that already called the hook (Rules 11, 18) — never a hook call inside the leaf |

**Worked example, in words:** at the page's orchestration boundary, the
page calls the permissions hook once to get two booleans — say, "can view
the list" and "can view a sensitive field." The first gates the list query
itself, so an unauthorized user never triggers a wasted fetch, and drives
an early access-denied render if the user can't view the list at all. A
"create" button further down is wrapped in the guard component, gated on
the create permission. The "can view sensitive field" boolean is passed
down as a plain prop into the row/card component, which stays fully
props-driven and never calls the permissions hook itself. One orchestration
point, three techniques, zero permission logic leaking into leaf
components.

For an early-stage app without a real backend yet, the session hook can be
a clearly labeled placeholder returning a fixed fake user — its permission
strings should be hardcoded literals rather than importing a module's
constants, since `core/` must never depend on a business module (Rule 16).
A real backend issues those same strings as token or session claims,
equally decoupled from the frontend's folder layout.

---

## Routing

The whole route tree is declared **once, as data**, in a central route
configuration — not scattered as individual route elements across the
tree. Each node carries:

- a **path**, sourced from the path registry, never a literal string;
- an **element**, the component to render;
- optional **children**, for nested routes;
- an optional **layout**, a shared shell attached once per group rather
  than re-wrapped by every page;
- optional **metadata** — a menu label, an icon, an order, a required
  permission — consumed both by navigation UI and by route guards.

Module pages contribute their own route fragment, which the central
assembler mounts. That's one declaration serving two consumers (routing and
navigation), so there's never a second, hand-maintained menu array to keep
in sync. Every page element is lazily loaded, so an unopened module
contributes near-zero bytes to the initial bundle.

### Route Tree at Enterprise Scale

One flat route file does not survive dozens of modules. Split it:

- Each module ships its own route fragment, using only its own pages and
  the shared path registry.
- The central route configuration becomes a pure assembler: it imports the
  fragments, mounts them under their group/layout parents, and keeps the
  catch-all route. It stays the only place the whole tree is visible.
- Route guards attach at the group/parent node covering a whole business
  area — one check per area, not repeated at every leaf.

---

## Centralized Route Paths

One single registry is the only source of literal route path strings.
Every navigation call and every link references that registry instead of a
literal string, so renaming or nesting a route becomes a one-line change
instead of a project-wide search. A shared not-found or access-denied
component linking to the literal root path is a documented, deliberate
exception (Rule 19) — it can't depend on the route registry (Rule 3), and
the root path is a safe, universal assumption rather than app-specific
knowledge.

---

## Multi-Tenancy & Active Scope (Optional)

*Skip this section entirely if the application is single-tenant.*

If the application scopes data by an active organization, workspace,
business unit, or similar concept, keep that "active scope" state in one
dedicated provider:

1. Bootstrap it from a single "who am I / what do I have access to"
   endpoint, persist the choice, and revalidate it when the app restores
   from a previous session.
2. On switch: validate the new scope against the backend, cancel in-flight
   queries, remove cached queries that aren't explicitly exempted (e.g.
   session-level queries), and let mounted observers refetch under keys
   that already embed the new scope (Rule 21) — never a blanket cache
   clear, which would also discard session data unnecessarily. Sync the
   active scope across open tabs so a switch in one tab doesn't leave
   another silently operating in the old scope.
3. Handle two edge cases explicitly: no accessible scope at all (route to
   a dedicated chooser/onboarding screen), and the active scope being
   revoked mid-session (show a clear "no longer authorized" state with a
   way to switch, rather than letting stale-scoped requests fail
   silently).

---

## Entity Workflows & State Machines (Optional)

*Relevant wherever a domain entity has an explicit lifecycle — orders,
tickets, approvals, invoices, applications — rather than being a simple
CRUD row.*

Model that lifecycle as an explicit state machine, not a loose status
string with ad hoc checks scattered through the UI. One shared pattern
should render, for an entity's current status: the actions available
(filtered by both permission and the transition table), the confirmation
each action requires, and any required justification for an override.

- Terminal statuses render read-only with a short explanation of why,
  rather than a disabled-and-unexplained button.
- Cancellation and reversal actions get honest copy about what will
  actually happen — a straightforward status flip for something still in
  draft is not the same operation as a compensating action for something
  already committed downstream, and the two should never share vague
  copy.
- Any override of a normal transition rule requires an explicit
  justification input, never a silent flag.
- Illegal transitions are ultimately refused by the backend, not just
  hidden by the frontend — surface that refusal inline at the action, then
  refresh the entity's actual state rather than trusting an optimistic
  guess.

---

## Design Tokens & Theming

Regardless of which UI library is chosen, maintain **one single source of
visual truth**: a token file defining the palette (brand colors, semantic
colors, neutrals, generic status tints), typography scale, spacing base and
control heights, corner radii per component category, shadows, and
transitions. Build the actual theme exclusively from those tokens plus
documented component-level overrides, so every component inherits
consistent spacing, radius, and control height by construction.

Features never hardcode a raw hex or pixel value — they reference theme
props or the token object (Rule 26). Status-to-color mappings (an
"approved" badge rendering green, say) live in the owning module's own
options/constants file and map onto generic semantic tint tokens, never
onto raw colors, so a future rebrand or dark-mode pass touches one file
instead of hundreds.

---

## Backend Integration Contract

Document your team's *actual* backend contract in one short, living
reference that every engineer reads once — this section describes the
shape that reference should take, not a specific contract, since the
specifics are genuinely backend-dependent. At minimum, capture:

- **Response envelope** — does every response share a common wrapper (a
  success flag, a machine-readable code, the payload, error details)? Are
  any endpoints exempt (binary downloads, a health check)?
- **Pagination strategy** — offset/page-based as the default, with any
  exceptions for append-only, ledger-style endpoints that use cursor-based
  pagination instead.
- **Filtering & sorting** — ideally the backend exposes filter metadata
  per resource so the frontend never hardcodes filter enums that can
  silently drift out of sync.
- **Cross-cutting headers** — auth token, tenant/scope headers, an
  idempotency-key convention, a CSRF-token echo — and confirmation that
  these live exclusively in the base API client (Rule 20), never attached
  ad hoc by a feature.
- **Error-code taxonomy** — stable, machine-readable codes the frontend can
  safely branch on, distinct from a free-text message meant only for
  display.
- **Lifecycle transition rules**, if entities have one — where they live,
  and what error code an illegal transition returns (see [Entity Workflows
  & State Machines](#entity-workflows--state-machines-optional)).

Keep a short "contract drift" log of any backend behavior the frontend has
learned about that isn't yet reflected in the official API docs, so the
next engineer doesn't rediscover it from a support ticket.

---

## Performance Budgets

Budgets are enforced by review and CI, not vibes:

- **Initial JS payload** — set an explicit gzip budget excluding framework
  vendor chunks (a couple hundred KB is a reasonable enterprise starting
  point; tune to your app), with vendor chunking kept cacheable across
  releases.
- **Per-route chunk** — one lazily loaded module page should ship well
  under a set budget including its own feature code; a bigger number
  usually means shared code failed to bubble up, or a heavyweight
  dependency snuck in.
- **No module is ever eagerly imported** outside its own route fragment.
- **Lists virtualize or paginate** beyond roughly 100 rendered rows (see
  [Forms & Tables at Enterprise Scale](#forms--tables-at-enterprise-scale)).
- **Compiler-assisted React era** — memoize only with profiler evidence;
  see the five questions in [Architecture Self-Check](#architecture-self-check).
- **Regression ritual** — before merging a module, compare chunk sizes
  against the previous release and investigate any significant growth.
  Automate the two budgets above as a CI gate rather than relying on
  someone remembering to check.
- **Icons and heavy dependencies** — import icons by name rather than
  through a barrel file that pulls in the whole set; keep charting, rich
  text, and PDF-generation libraries in their own lazy-loaded chunk, not
  the main bundle.

---

## Naming Conventions & Code Style

All application-created names below are in **English by default**. See
[Appendix A](#appendix-a--french-language-naming-convention-french-client-projects-only)
for the French-language variant used only on French client projects.

### General Rules

- Prefer descriptive names over abbreviations.
- Keep well-known ecosystem abbreviations as-is: `API`, `UI`, `URL`, `ID`,
  `DTO`.
- Never reuse the same word for two different concepts.
- Avoid vague catch-all names: `data`, `stuff`, `temp`, `thing`, `helper`,
  `common`.

### Folder Naming Rules

- Ordinary folders use `kebab-case` — `customer-list`, `create-customer`.
- Group folders use parentheses: `(dashboard)`, `(marketing)`. A group
  folder name doesn't have to be a business term — a folder of shared
  reference/example content can be grouped just as legitimately as a
  layout concern.
- Feature and module names describe a bounded context: `customers`,
  `billing`, `scheduling` — never `misc` or `other`.
- Every module exposes an index/gateway file.

### File Naming Rules

- **Components:** `PascalCase.tsx` — `CustomerCard.tsx`,
  `CustomerForm.tsx`, `PermissionGuard.tsx`.
- **Hooks:** `use` + PascalCase — `useCustomerFilters.ts`,
  `useCreateCustomer.ts`, `usePermissions.ts`.
- **Mappers / data-access functions:** named after the action they
  perform — `mapCustomerDtoToCustomer.ts`-style logic inside
  `customerMapper.ts`, `listCustomers.ts`, `createCustomer.ts`.
- **Test files:** subject-first, matching the project's standard test
  suffix — `CustomerTableSection.test.tsx`, `permissionEngine.test.ts`.
  Test *description strings* read as documentation of behavior for any
  engineer and are surfaced in CI logs, regardless of the project's
  identifier-naming-language policy.
- **Infrastructure code without a "feature" skeleton** (most of `core/`)
  colocates its test next to the source file rather than using a `tests/`
  subfolder; `feature/tests/` is reserved for actual features under
  `modules/`.

### Class and Type Naming Rules

- `PascalCase`, no `I`/`T` prefixes.
- Components named for what they render: `CustomerCard`, `CustomerForm`.
- Provider components suffixed `Provider` (or your framework's own
  convention).
- Error boundaries named for their role.
- Domain entities named after the business concept: `Customer`.
- DTOs suffixed `Dto`: `CustomerDto`, `CreateCustomerRequestDto`.
- Frontend contract types follow one consistent pattern:
  `<Concept>Response` / `<Action><Concept>Request` —
  `CustomerResponse`, `CreateCustomerRequest`.
- Permission-related types split into an opaque string alias at the engine
  level (`Permission`, in `core/permissions/`) and a module-owned literal
  union at the vocabulary level (`CustomerPermission`, in
  `module/domain/`).

### Method and Function Naming Rules

- **Queries:** `get`, `list`, `find` — `listCustomers`.
- **Mutations:** `create`, `update`, `delete` — `createCustomer`.
- **Transformations:** `map`, `build`, `format`, `filter`, `validate` —
  `mapCustomerDtoToCustomer`, `formatDate`, `filterCustomersBySearch`,
  `validateEnvironment`.
- **Boolean checks:** `is`, `has`, `can`, `should` — `hasPermission`,
  `hasAllPermissions`, `hasAnyPermission`.
- **Prop callbacks:** the framework's own convention — `onSubmit`,
  `onCancel`, `onRetry`.

### Variable Naming Rules

- `camelCase`.
- Booleans read as a state or a question: `isLoading`, `canEdit`,
  `hasErrors`, `isModalOpen`.
- Collections are plural nouns: `customers`, `filteredCustomers`.
- Identifiers keep the recognized `Id` suffix: `customerId`.
- Timestamps follow one consistent convention applied everywhere —
  `createdAt`, `updatedAt` — never invented per module.

### Enum and Constant Naming Rules

- Constants: `UPPER_SNAKE_CASE` — `DEFAULT_PAGE_SIZE`,
  `SEARCH_DEBOUNCE_MS`.
- Permission keys are the backend's own catalog codes, used **verbatim**,
  grouped under one `UPPER_SNAKE_CASE` constant object per module in that
  module's domain layer (`CUSTOMER_PERMISSIONS`, …) — never invented
  independently on the frontend.
- Route paths follow whatever casing the URLs are actually served in,
  collected in the single path registry.
- When an enum's values are contractual (the backend sends a literal
  string like `"ADMIN"`), keep those values verbatim and put the
  human-readable label in a separate, module-owned mapping — so the wire
  format and the display text can change independently.

### i18n Key Naming Rules

- Translation objects use camelCase keys.
- Keep keys scoped to the feature or module that owns the copy, unless a
  string is genuinely reused everywhere — in which case it earns a place
  in the shared i18n layer (Rule 17).

### Naming Anti-Patterns to Avoid

- Vague filler words as variable names: `data`, `thing`, `temp`.
- Dumping-ground folder names: `helpers`, `common`, `utils` at the top
  level.
- Mixing two naming languages in the same file (relevant mainly on French
  client projects — see Appendix A).
- Copying backend field names straight into UI code without going through
  the mapper layer.

### Naming Priority Order

When a name is genuinely unclear, resolve in this order:

1. Business meaning
2. Layer responsibility
3. Public API clarity
4. Technical convention
5. Conciseness

---

## Testing Organization

### Placement rule — one table, no ambiguity

| Where does the file live? | Where does the test go? | Rationale |
| --- | --- | --- |
| `core/`, `shared/`, `api/base/`, `app/`, `routes/` | Colocated — same directory as the source | Infrastructure is stable, few files per folder; colocation makes discovery trivial. |
| `modules/<m>/api/` and `modules/<m>/domain/` | Colocated — same directory as the source | Module-level infrastructure (mappers, entities, permission keys) follows the same rationale. |
| `modules/<m>/features/<f>/` (any sub-layer) | `feature/tests/` subfolder | Features grow to many files across `application/`, `components/`, `sections/`, `validation/`, `hooks/` — a dedicated tests folder prevents clutter. |

**Decision shortcut:** infrastructure for the module (`api/` or `domain/`
at the module root) → colocate. Part of a feature (anywhere under
`features/<f>/`) → `features/<f>/tests/`.

### Additional rules

- Test file names match the unit under test.
- Test description strings read in English, since they're documentation
  read in CI logs by anyone on the team — this stays true even on a
  French-named codebase (see Appendix A); the code exercised inside the
  test follows the project's chosen identifier-naming language.
- Run the project's standard single-pass test script routinely, a
  watch-mode script while developing, and a coverage script before merging
  a module.
- CI runs lint, typecheck, test-with-coverage, and build on every push and
  pull request.
- Prefer a lightweight, dependency-free approach to mocking network calls
  at the fetch level over a heavier mocking framework, with handlers
  grouped by domain and shared envelope fixtures — the goal is fast,
  deterministic tests without a second HTTP-mocking system to maintain.
  Bespoke per-test fetch stubs are still banned; use the shared harness's
  override mechanism instead.
- Add a thin layer of end-to-end smoke tests covering the handful of flows
  that would be genuinely bad to ship broken (login, the core workflow the
  product exists for), run on a schedule and before releases rather than
  on every commit if they're slow.

### Ownership matrix (enterprise scale)

| Layer | Owner | Required tests | Category |
| --- | --- | --- | --- |
| `domain/` (pure logic) | Owning module team | Exhaustive unit tests — the cheapest, highest-value layer | Unit |
| `application/` hooks | Owning module team | Hook behavior: states, invalidation, optimistic rollback | Unit/integration, via one shared fetch-mocking helper introduced when the first feature needs it |
| `sections/`, `components/` | Owning module team | Render states including loading/error/empty; props-driven permission booleans | Component |
| Page composition | Owning module team | One smoke test: renders, gates by permission | Component |
| `core/`, `shared/`, `api/base/` | Platform owner | Colocated unit tests, high bar — depended on by everyone | Unit |
| Route wiring | Platform owner | Type-checked by the config itself; dedicated tests only if guards get complex | — |

A module isn't "done" until its test pyramid is green and its `domain/` and
`application/` layers show meaningful coverage; a numeric UI-coverage
target is deliberately not set — it invites brittle tests at enterprise
scale.

---

## Tooling & Automation

Automation enforces the rules above instead of relying purely on code
review:

| Tool | Enforces |
| --- | --- |
| Linter custom rules/overrides | No feature-to-feature imports (Rule 1); `shared/` never depends on `modules/`, `app/`, `routes/`, or `core/` (Rule 3); a module's internal DTO/endpoint layer never leaks outside it (Rule 14); no `misc`/`helpers`/`common`/`other` folders (Rule 13). |
| The config-validation module | Validates environment variables against a schema, and never throws at startup (see [Startup & Configuration Safety](#startup--configuration-safety)). |
| Pre-commit hook + staged-files runner | Runs lint and format only on staged files before every commit. |
| Commit-message hook + convention checker | Enforces a consistent commit-message convention, in English. |
| CI pipeline | Lint, typecheck, test-with-coverage, build — on every push/PR. Plus a bundle-size budget gate and scheduled end-to-end smoke tests once the app is large enough to warrant them. |
| Fetch-mock harness | A zero-dependency fetch router for all network tests, with handlers grouped by domain and shared fixtures. |
| Automated dependency updates | Scheduled, grouped update PRs — reviewed, not auto-merged for anything touching the runtime. |

---

## Documentation Governance & Definition of Done

### Keeping this document true

- Structural changes to the architecture go through a short decision
  record (context / decision / consequences, one page) rather than
  silently drifting from what this guide says. A reviewer should ask for
  one if a PR changes a documented rule without one.
- Assign ownership of the platform layers (`core/`, `shared/`, `api/base/`,
  `routes/`) to a named owner or small group once more than one team
  touches the codebase; let each business module belong to its owning
  team. Introduce formal ownership tooling only once a second team joins
  — not before.
- This document describes *invariants*; a separate contributing guide can
  carry the how-to. When the two disagree, this document wins until the
  contributing guide is fixed to match.

### Definition of Done for a new module

- [ ] Follows the Module & Feature Convention; the gateway file exports
      only consumed symbols.
- [ ] Routes fragment added to the central route configuration; pages
      lazy-loaded; navigation metadata provided; permission guard applied
      at the appropriate group level.
- [ ] Query key factory created; staleness tiers chosen deliberately;
      mutations invalidate narrowly.
- [ ] Permission keys defined in the module's own domain layer; `core/`
      untouched.
- [ ] List/detail views implement all four states; the error-handling
      taxonomy is respected.
- [ ] Standard lint, typecheck, test, and build scripts all pass.
- [ ] Bundle impact checked against the performance budgets.
- [ ] Any cross-module dependency uses a sanctioned mechanism, or is
      explained by a decision record.

---

## Dead Code Hygiene

A generic-sounding constant, type, or copy string with **no current
consumer** is dead code, not a head start — it costs a future reader time
figuring out whether it's safe to touch, and its presence implies a usage
that doesn't actually exist anywhere.

**Rule of thumb:** if you're adding something "because a future feature
will probably need it," it belongs in that future feature's own branch
when it actually arrives — not preemptively promoted into `shared/` or
`module/domain/` today. Promote code upward only once a second real
consumer appears (the same reasoning as Rule 2).

---

## Architecture Self-Check

Run through this after any non-trivial change — before opening a PR, not
just once at the end of a big feature.

### Naming & language
- [ ] Every new component/function/variable/file/folder name follows the
      project's chosen identifier language (English by default; French
      only per Appendix A on French client projects), consistently within
      each file.
- [ ] Every new comment, and every doc file touched, reads in English —
      this stays true even on a French client project.
- [ ] Every new commit message follows the team's convention, in English.
- [ ] Every new end-user-facing string matches the application's actual
      target language(s).

### Layering & imports
- [ ] Nothing in `shared/` imports from `modules/`, `app/`, `routes/`, or
      `core/` (run the linter — this is enforced, not just documented).
- [ ] No feature imports another feature directly — only via
      `module/shared/` or `module/domain/`.
- [ ] A module's internal DTO/endpoint layer is only imported from inside
      that module itself.
- [ ] Nothing in `core/` references a specific business concept (a
      permission's meaning, a domain entity, a feature name).
- [ ] Every module/feature touched still has an accurate, narrow gateway
      file — no wildcard export, nothing exported that isn't actually
      consumed from outside.
- [ ] Any module-to-module dependency uses a sanctioned mechanism (gateway
      import, URL navigation, query-cache invalidation, promotion to
      `shared/`) — downstream direction only.

### State & data
- [ ] Server data is only ever held by the data-fetching library — no
      component-state or store copy of a query result "for convenience."
- [ ] Any new query/mutation uses the single base HTTP client, not a new
      HTTP library.
- [ ] Any new list/detail view has three distinct states — loading, error
      (with a retry action), and successfully empty — not two. A silently
      empty page on a failed request is exactly the bug this architecture
      is built to avoid.
- [ ] Any code that reads the raw environment directly, instead of going
      through the config-validation module, is a regression — fix it
      before merging.
- [ ] New queries use the owning module's key factory; mutations
      invalidate the narrowest correct keys; the staleness tier is chosen
      deliberately.

### Permissions
- [ ] Any new page-level access rule uses the permissions hook (for the
      boolean, e.g. to also gate a query) or the guard component (for a
      JSX subtree) — not an inline role/permission comparison in JSX.
- [ ] The permission's key is defined in the owning module's domain layer,
      not hardcoded inline and not added to `core/permissions/`.
- [ ] Permission checks happen at the orchestration boundary and are
      threaded down as props to any purely presentational component that
      needs them — not called from inside a leaf component.

### Routing
- [ ] Every new route is added to the central route configuration, and
      every navigation call references the path registry rather than a
      literal string.
- [ ] New pages are lazy-loaded in the route config, consistent with the
      existing pattern.
- [ ] A new reusable layout wrapping several routes is attached through
      the route configuration's layout mechanism, not by each page
      re-wrapping itself.
- [ ] New module pages are contributed via the module's own route
      fragment and are lazy-loaded; navigation entries come from route
      metadata, not a parallel menu array.

### React correctness & performance

A modern React version, especially with compiler-assisted memoization,
changes what "correct" React code looks like compared to older eras.
Reflexively reaching for manual memoization or an effect because that used
to be the safe default is itself a code smell to look for now, not a
virtue. Two failure modes pull in opposite directions — check both:

- **Under-using React:** an effect doing what an event handler or a plain
  computed value should do (harder to trace, an extra render pass, prone
  to firing at the wrong time).
- **Over-using React:** manual memoization sprinkled onto cheap
  computations and small components "just in case," adding real
  complexity (a dependency array to keep correct, a second thing to reason
  about) for zero measurable benefit.

**The five questions** — ask these of every manual memoization call or
effect you touch or add:

1. Is it actually necessary?
2. What problem does it solve?
3. Could the same behavior be implemented more simply (a plain
   computation, a direct event handler, state adjusted during render)?
4. Does the current React version — or a compiler, where enabled — already
   handle this automatically?
5. Is there a measurable performance reason to keep it (a profiler result,
   a genuinely expensive computation, a large list), or is this guesswork?

If the honest answer to #1 is "not sure," that's the signal to simplify,
not to leave it in "to be safe." Before reviewing individual files, sweep
the codebase for every effect and every manual memoization call outside
test files, and run each one through the checklist below — the point is to
cover the whole codebase in one pass, not just whatever file happens to be
open.

**Effect audit** — for each occurrence:
- [ ] It synchronizes with something genuinely *external* to React (the
      DOM, a subscription, a timer, a browser API, a third-party widget)
      — not with another piece of this app's own state.
- [ ] It is not computing **derived state** — that belongs in a plain
      computed value during render.
- [ ] It is not reacting to a state change that was itself caused by a
      user event in the same component tree — call the follow-up logic
      directly in that event handler instead.
- [ ] It is not just replicating a network request the data-fetching
      library would already own (Rule 7).
- [ ] Its dependency list is complete and honest — a suppressed lint
      warning here is usually hiding one of the cases above, not a false
      positive.

**Derived state & duplicated state audit:**
- [ ] No local state holds a value that could instead be a computed value
      derived directly from props/other state during render.
- [ ] No two pieces of state can go out of sync with each other because
      one is really a function of the other — collapse them to one source
      of truth plus a computed value.
- [ ] Where a computed value is genuinely reused across many renders with
      no cheap alternative, memoization is a conscious, documented choice
      — not a reflex.

### Backend contract & scope
- [ ] The response envelope is parsed only in the base API client;
      features branch on typed error codes, never raw response bodies.
- [ ] No manual attachment of cross-cutting headers (auth, tenant/scope,
      CSRF, idempotency) outside the base client and its centrally
      registered providers.
- [ ] Query keys embed the active scope via the module factory when the
      app is multi-tenant; no bare full-cache-clear anywhere.
- [ ] List filters render from backend-provided metadata where available
      — no hardcoded filter enums for filtering logic.
- [ ] Mandated creations on financial/critical resources carry an
      auto-generated idempotency key; replays surface as a notice.
- [ ] Money/quantity fields use string-preserving inputs; zero
      client-side float math on amounts; totals displayed from server
      fields.
- [ ] Entity/document actions derive from the owning family's status map
      through the shared workflow-action component; terminal statuses are
      read-only; any forced override carries a required justification.
- [ ] Permission strings are imported from module constants only — no
      raw literals in JSX.
- [ ] Optimistic updates are limited to the benign-mutation cases in
      Rule 24.
- [ ] No hardcoded colors/padding/radii in features — theme props or the
      token object only; status-to-tint mappings live in each module's
      options file.

---

## Appendix A — French-Language Naming Convention (French Client Projects Only)

> **This appendix applies only when the delivered product is for
> French-speaking end users, and the team has additionally chosen to
> reflect that by writing business-domain *code identifiers* in French —
> not just user-facing text.** If that isn't this project, **delete this
> appendix**. Nothing else in this guide references it, and every example
> elsewhere in this guide is already in English.

### Language Policy

| What | Language | Why |
| --- | --- | --- |
| Code identifiers — components, functions, variables, hooks, types, files, folders, query keys, test *subjects* | **French** | Business-domain vocabulary matches how the client and end users actually talk about their own business. |
| Code comments, doc comments | **English** | Comments explain *why* to any engineer who joins the team, regardless of their proficiency in French. |
| Markdown documentation (README, this file, a contributing guide, a changelog) | **English** | Same reasoning as comments — documentation is a developer-facing artifact, read by any engineer on any future project. |
| Git commit messages, PR titles/descriptions | **English** | Keeps project history searchable and reviewable for any contributor. |
| Test *description strings* (the argument to a test's name) | **English** | They function as documentation of behavior, read by any engineer and surfaced in CI logs — see [Naming Conventions](#naming-conventions--code-style). |
| End-user-facing text (UI labels, buttons, validation messages, emails) | **French** | The application is used by French end users — the product itself must speak their language. |
| Framework- or library-required identifiers (a provider component's name, a config filename) | **Unchanged** | These are contractual names owned by the library or tool, not by this codebase. |

**In short:** open any source file and both the *symbols* and the *prose
around them* (comments, docs, commit messages, test descriptions) read in
English. Only end-user-facing text and the business-domain vocabulary of
the code itself are French.

### Technical Terms Glossary

The following universal technical terms stay in **English** even inside a
French-named codebase — they're industry vocabulary, not French-domain
business concepts. Extend this list with the specific tool names your
stack actually uses.

`API` · `UI` · `URL` · `ID` · `DTO` · component · hook · query · mutation ·
cache · store · module · feature · layout · page · context · schema ·
field · state · props · type · error · boundary · layer · wrapper ·
config · import · export · test · coverage · CI · commit · Git ·
TypeScript · React · fetch · chunk · bundle · memo · Promise · async ·
provider · token · path · routes · permission · guard · toast

### Folder Naming Rules

- Ordinary folders use `kebab-case`, in French — `liste-clients`,
  `creer-client`.
- Group folders use parentheses regardless of language: `(exemple)`,
  `(tableau-de-bord)`.
- Feature and module names describe a bounded context in French:
  `utilisateurs`, `facturation`, `rendez-vous` — never `divers` or
  `autre`.

### File Naming Rules

- **Components:** `PascalCase.tsx` in French — `CarteUtilisateur.tsx`,
  `FormulaireUtilisateur.tsx`.
- **Hooks:** `use` + French PascalCase — `useFiltreUtilisateurs.ts`,
  `useCreerUtilisateur.ts`.
- **Mappers / data-access functions:** named after the French verb for the
  action — `obtenirListeUtilisateurs.ts`, `creerUtilisateur.ts`.
- **Test files:** `*.test.ts(x)`, subject first, in French —
  `TableauUtilisateurs.test.tsx` — but the description string inside stays
  English (see Language Policy above).

### Class and Type Naming Rules

- `PascalCase`, no `I`/`T` prefixes.
- Domain entities: `Utilisateur`.
- DTOs: `<Concept>Dto` — `UtilisateurDto` (`Dto` stays as the recognized
  ecosystem abbreviation even in French code).
- Frontend contracts: `<Concept>Reponse` / `<Action><Concept>Entree` —
  `UtilisateurReponse`, `CreerUtilisateurEntree`.

### Method and Function Naming Rules

French verbs, `camelCase`:

- **Queries:** `obtenir`, `lister`, `trouver` — `obtenirListeUtilisateurs`.
- **Mutations:** `creer`, `mettreAJour`, `supprimer` — `creerUtilisateur`.
- **Transformations:** `mapper`, `construire`, `formater`, `filtrer`,
  `valider` — `mapperUtilisateurDtoVersUtilisateur`, `formaterDate`.
- **Boolean checks:** `est`, `a`, `peut`, `doit` — `aPermission`.
- **Prop callbacks** keep the React `on` prefix as-is; the rest of the name
  is French — `onSoumission`, `onAnnuler`.

### Variable Naming Rules

- `camelCase`, French.
- Booleans: `est`, `a`, `peut`, `doit`, `en` — `peutConsulter`,
  `chargementEnCours`, `modaleOuverte`.
- Collections: plural French nouns — `utilisateurs`,
  `utilisateursFiltres`.
- Identifiers keep the `Id` suffix as-is: `utilisateurId`.
- Dates: `creeLe`, `misAJourLe` (French equivalents of `createdAt` /
  `updatedAt`).

### Enum and Constant Naming Rules

- Constants: `UPPER_SNAKE_CASE`, French words — `TAILLE_PAGE_PAR_DEFAUT`.
- Permission keys are the backend's own codes, used verbatim regardless of
  naming language, grouped under an `UPPER_SNAKE_CASE` object in the
  owning module — `PERMISSIONS_UTILISATEURS`.
- Enum-like values that are contractual with the backend keep their actual
  coded value (`"ADMIN"`), while their **display label** is French, in a
  separate module-owned mapping (`"ADMIN"` → `"Administrateur"`).

### i18n Key Naming Rules

- Translation objects use French camelCase keys **and** French string
  values — e.g. `libellesUtilisateurs.titrePage`. Both the key and the
  value are French here, consistent with "code identifiers are French."
- Keep keys domain-scoped, not generically shared, unless they truly
  belong in the shared i18n layer.

### Naming Anti-Patterns to Avoid

- `donnees`, `truc`, `chose`, `temp` as variable names.
- `aide`, `aides`, `commun` as dumping-ground folder names.
- Mixing French and English identifiers in the same file.
- Copying backend DTO field names directly into UI code without going
  through the mapper layer.

### Naming Priority Order

1. Business meaning, in French
2. Layer responsibility
3. Public API clarity
4. Technical convention
5. Conciseness

---

Everything else in this guide — folder layout, layering rules,
data-fetching patterns, testing organization, performance budgets — applies
identically whether a project uses this appendix or not. Only the
vocabulary changes.
