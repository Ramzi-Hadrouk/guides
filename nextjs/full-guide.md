# Enterprise Next.js Architecture Guide

*A stack-flexible, principle-driven architecture reference for building
Next.js (App Router) applications that stay scalable, maintainable, and
fast as an enterprise codebase, its team, and its business domain all
grow.*

This is the **Next.js counterpart** to the general Enterprise React
Architecture Guide — same goals, the same numbered rules (extended with a
few that Next.js's rendering model genuinely requires), the same section
order. Sections that a rendering framework doesn't change are kept short
and consistent with the base guide; sections Next.js genuinely changes —
rendering, routing, data fetching, forms, error boundaries, startup
safety, performance, testing — are rewritten in full below. Where the two
guides overlap, prefer this one for a Next.js project; nothing here
requires reading the base guide first.

## How to Use This Guide

- Adopt as much or as little as fits your project. [What Must Be
  Centralized](#what-must-be-centralized) and [Architectural
  Rules](#architectural-rules) carry the most weight — every other section
  supports one of those two.
- This guide assumes the **App Router** (not the older Pages Router) and
  React Server Components. If a project is still on the Pages Router, the
  layering and naming sections still apply, but [Rendering
  Strategy](#rendering-strategy--server-vs-client-components) and
  [Routing](#routing) do not — treat this guide as App-Router-specific.
- This guide is written entirely in **English**, and every example
  identifier in it is English, so it is ready to use as-is on any project.
- The one exception is **[Appendix A](#appendix-a--french-language-naming-convention-french-client-projects-only)**,
  which documents a French-language naming convention for code
  identifiers. It applies **only** to projects delivered to
  French-speaking end users where the team has also chosen to write
  business-domain *code* in French, not just user-facing text. **If
  that isn't this project, delete Appendix A outright.**
- Everything is a starting point, not scripture. Where this guide's
  default disagrees with a constraint your project actually has, write
  down the deviation (see [Documentation
  Governance](#documentation-governance--definition-of-done)) and move on.

---

## Contents

1. [Core Architectural Goals](#core-architectural-goals)
2. [What Must Be Centralized](#what-must-be-centralized)
3. [Recommended Technology Stack](#recommended-technology-stack)
4. [Project Structure](#project-structure)
5. [Rendering Strategy — Server vs. Client Components](#rendering-strategy--server-vs-client-components)
6. [Module & Feature Convention](#module--feature-convention)
7. [Scaling Model — From One Module to an Enterprise System](#scaling-model--from-one-module-to-an-enterprise-system)
8. [Cross-Module Communication](#cross-module-communication)
9. [Route Grouping](#route-grouping)
10. [Layer Responsibilities](#layer-responsibilities)
11. [Architectural Rules](#architectural-rules)
12. [Startup & Configuration Safety](#startup--configuration-safety)
13. [HTTP & Data Fetching](#http--data-fetching)
14. [Data Layer at Scale](#data-layer-at-scale)
15. [Forms & Tables at Enterprise Scale](#forms--tables-at-enterprise-scale)
16. [Error-Handling Taxonomy](#error-handling-taxonomy)
17. [Permissions & Conditional UI](#permissions--conditional-ui)
18. [Routing](#routing)
19. [Centralized Route Paths](#centralized-route-paths)
20. [Multi-Tenancy & Active Scope (Optional)](#multi-tenancy--active-scope-optional)
21. [Entity Workflows & State Machines (Optional)](#entity-workflows--state-machines-optional)
22. [Design Tokens & Theming](#design-tokens--theming)
23. [Backend Integration Contract](#backend-integration-contract)
24. [Performance Budgets](#performance-budgets)
25. [Naming Conventions & Code Style](#naming-conventions--code-style)
26. [Testing Organization](#testing-organization)
27. [Tooling & Automation](#tooling--automation)
28. [Documentation Governance & Definition of Done](#documentation-governance--definition-of-done)
29. [Dead Code Hygiene](#dead-code-hygiene)
30. [Architecture Self-Check](#architecture-self-check)
31. [Appendix A — French-Language Naming Convention (French Client Projects Only)](#appendix-a--french-language-naming-convention-french-client-projects-only)

---

## Core Architectural Goals

Unchanged from the base guide — every recommendation here still serves one
of these four goals, and when two pieces of advice seem to conflict,
resolve in favor of whichever matters more for the change at hand.

- **Scalability** — new modules, features, and teams add code without
  adding coupling.
- **Maintainability** — a change in one place has a small, predictable
  blast radius, and deleting a feature is safe (Rule 9).
- **Performance** — the app stays fast as it grows, by enforced budget,
  not by luck (see [Performance Budgets](#performance-budgets)) — Next.js
  gives this goal a genuine head start via server rendering and
  Server Components, which this guide leans on deliberately rather than
  accidentally.
- **Safety** — a bad request, an invalid config value, or a bug in one
  corner of the app degrades gracefully instead of producing a blank
  screen or corrupted data (see [Startup & Configuration
  Safety](#startup--configuration-safety) and [Error-Handling
  Taxonomy](#error-handling-taxonomy)).

The point of folder structure is still never organization for its own
sake — it's **clear ownership**, **strict boundaries**, **low coupling**,
**easier refactoring**, and **safe deletion**.

---

## What Must Be Centralized

Unchanged in spirit from the base guide, with a few rows adapted to what
Next.js actually needs centralized.

| Concern | Centralize in | Why one place |
| --- | --- | --- |
| Environment & configuration validation | One config module, split by build-time vs. request-time (see [Startup & Configuration Safety](#startup--configuration-safety)) | So an invalid deploy fails predictably in one spot instead of surfacing as a mystery bug deep in a feature. |
| Client-exposed vs. server-only environment variables | The same config module, with the exposure boundary enforced (Rule 28) | A server secret leaking into a `"use client"` file ships it to every visitor's browser. |
| HTTP transport & response parsing | One base API client | So the backend's response shape is understood in exactly one layer, whether it's called from a Server Component, a Route Handler, or a Client Component — see [HTTP & Data Fetching](#http--data-fetching). |
| Base URL resolution per runtime | The base API client | Server-side calls may need an internal address; client-side calls need a public one — resolving this ad hoc per call site is how the two quietly drift. |
| Authentication/session state & token lifecycle | One session module, with a server-side reader and a client-side hook | Token refresh races and logout edge cases are hard enough to get right once. |
| Permission *evaluation* engine | One core module | The engine stays a pure, business-blind string comparator; the *vocabulary* stays distributed per module — see [Permissions & Conditional UI](#permissions--conditional-ui). |
| Route path strings & dynamic-segment builders | One path registry | Renaming or nesting a route becomes a one-line change instead of a project-wide search. |
| Navigation metadata (menu label/icon/order/permission) | One aggregated registry, fed by per-module exports | Next.js's file system gives you the route tree for free, but not a nav menu — see [Routing](#routing). |
| Design tokens / visual theme | One token file + a generated theme | A rebrand, a dark mode, or a UI-library swap touches one file instead of hundreds — see [Design Tokens & Theming](#design-tokens--theming). |
| Error-handling taxonomy | One mapping of error category → UI treatment | So "how do we show this failure" is a lookup, not a fresh decision per feature — see [Error-Handling Taxonomy](#error-handling-taxonomy). |
| Logging / observability sink | One logging abstraction | So redaction, formatting, and the eventual choice of logging backend are decided once. |
| Active tenant/business scope *(if multi-tenant)* | One scope provider, backed by a cookie readable on both server and client | Prevents one tenant's cached data from ever leaking into another tenant's view — see [Multi-Tenancy & Active Scope](#multi-tenancy--active-scope-optional). |
| Query-key / cache-tag factory pattern | One factory per module, following one shared shape | Keeps cache invalidation predictable whether it's a client query key or a server revalidation tag — see [Data Layer at Scale](#data-layer-at-scale). |
| Revalidation policy (time-based or tag-based) | Declared once per data-fetching function, close to the data | So a mutation always knows exactly what to invalidate, instead of an inconsistent option scattered across call sites. |
| Generic, truly shared i18n copy | One shared i18n layer | Ownership of most copy stays distributed (feature → module → shared) — see Rule 17 — but the generic tier belongs in exactly one file. |
| Feature-flag evaluation *(if used)* | One provider | So a flag's value can never disagree with itself between two parts of the same page. |

Everything **not** on this list is deliberately *not* centralized: business
logic, permission vocabulary, feature-local copy, and feature-local
validation schemas live next to the code that owns them (see [Module &
Feature Convention](#module--feature-convention)).

---

## Recommended Technology Stack

Every row below is a default with a reason, not a mandate — swap freely as
long as the underlying principle survives the swap.

| Category | Recommended default | Why | Reasonable alternatives |
| --- | --- | --- | --- |
| Framework / rendering | Next.js, App Router, React Server Components by default | Server-rendered HTML, streaming, and per-segment loading/error boundaries out of the box, and a large fraction of the app can ship zero client JavaScript by construction — directly serves the Performance goal. | The Pages Router remains viable for teams not ready for the Server Components data model, but a new enterprise project should default to the App Router. Remix or TanStack Start are reasonable alternatives with a similar server-first philosophy if Next's specific conventions don't fit. |
| Language | TypeScript, strict mode on | The layering and boundary rules in this guide lean on the compiler to catch accidental coupling, including the server/client boundary itself. | Not seriously recommended to skip at this scale. |
| UI component library / design system | Pick one and commit | As long as [Design Tokens & Theming](#design-tokens--theming) is followed, swapping the library later only touches the theming layer. | Any mature library, or a utility-CSS + headless-component setup, works equally well — the latter pairs especially naturally with Server Components, since plain CSS needs no client runtime at all. |
| Data fetching — server | Native `fetch` (or a direct data-access/ORM call) inside `async` Server Components and Route Handlers | No client-side network waterfall, no loading spinner for data that could already be in the initial HTML. | — |
| Data fetching — client | TanStack Query, scoped to Client Components that genuinely need client-driven caching, polling, or optimistic mutations | Still the strongest option once data is truly client-owned. | A lighter alternative with a smaller API surface if the client-side surface area is small. |
| Low-level HTTP | A thin, envelope-aware wrapper around `fetch`, extended to resolve its base URL per runtime and thread Next's cache/revalidate options through | One low-level layer, callable from Server Components, Route Handlers, Server Actions, and Client Component query hooks alike. | A promise-based HTTP client is a fine swap for the same reasons as the base guide — a real need for interceptors, upload-progress events, or old-browser support. |
| Mutations | Server Actions by default; Route Handlers for endpoints called from outside the app itself (webhooks, a mobile client) | Server Actions collapse validation, the mutation, and revalidation into one server-side place per form — see Rule 29. | A Client Component + a mutation hook remains reasonable for mutations that need richer client-side interactivity — see [Forms & Tables at Enterprise Scale](#forms--tables-at-enterprise-scale). |
| Validation | A schema-validation library whose inferred types double as form/DTO types, shared between client-side validation and the server-side check that actually enforces it | Client and server validation can no longer quietly drift, since they read the same schema. | — |
| Global client state | Nothing, by default; server state plus component-local state covers most screens | Applies only within Client Component subtrees — a global store added "just in case" is unused complexity, doubly so here since it can't help Server Components at all. | A lightweight store once a genuine cross-cutting Client need appears. |
| Routing | Next.js's own file-system router — see [Routing](#routing) | No separate router library needed; route-level code splitting is automatic. | — |
| Forms | The framework's own form-status hooks (`useActionState`/`useFormStatus`) paired with a Server Action, or a client mutation hook for richer interactivity | Matches whichever of the two mutation paths above the form actually needs. | — |
| Testing | A Vitest-based runner + a user-behavior-focused component testing library for Client Components and pure functions; a modern cross-browser tool for end-to-end coverage | Server Components are async and don't render well under a component-testing library — end-to-end coverage carries relatively more weight here. See [Testing Organization](#testing-organization). | Any well-maintained equivalent in the same category. |
| Linting / formatting | Next's own lint configuration as a base layer, extended with the same custom, path-based architectural rules as the base guide | The cross-feature-import ban, the shared-layer dependency direction, and the new server/client import boundary (Rule 28) only hold long-term if a machine enforces them on every change. | — |
| Git hooks & commit convention | Same as the base guide — a pre-commit hook on staged files, a commit-message convention. | — | — |
| CI | Whatever your organization already standardizes on, running lint, typecheck, test-with-coverage, and build on every push/PR, plus a route-segment size comparison from the build output. | The specific provider is not architecturally significant. | GitHub Actions, GitLab CI, Azure DevOps, and so on. |
| Images & fonts | The framework's built-in image and font optimization, used by convention instead of a hand-rolled `<img>` tag or a manually linked web font | Low-effort, high-impact performance wins that are effectively free once adopted as a rule. | — |

---

## Project Structure

> **Naming note:** unlike the base guide, `app/` is a **reserved Next.js
> directory name** for the router itself. The base guide's
> composition-root layer (providers, the top-level error boundary) is
> renamed `providers/` here to avoid the collision.

- `.env.local` (untracked — real secrets), `.env` / `.env.example`
  (tracked — shared, non-sensitive defaults) — client-exposed values are
  prefixed per Next.js convention so the exposure boundary is visible at
  the variable name itself (Rule 28). All of them are read exclusively
  through the config-validation module — nothing else in the codebase
  reads the raw environment directly.
- `src/`
  - `app/` — Next.js's reserved routing root. Route segments only:
    `layout.tsx`, `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`,
    `template.tsx`, Route Handlers (`route.ts`), route groups
    (`(dashboard)`, `(auth)`, `(marketing)`), dynamic segments. Every file
    here is intentionally thin — it imports and renders a real
    implementation from `modules/` or `layouts/`, and sets metadata;
    it contains no business logic of its own.
  - `middleware.ts` — edge-level cross-cutting checks: auth redirect, an
    active-scope cookie check, locale. Business-blind, same spirit as
    `core/` (Rule 16).
  - `providers/` — the composition-root Client Component tree: global
    providers (theme, the data-fetching client), the top-level
    client-side error boundary (supplementing `global-error.tsx`).
    Mounted once from the root `app/layout.tsx`.
  - `bootstrap/` — startup logic that runs once per server instance
    (e.g. startup logging registration).
  - `core/` — the technical, business-agnostic foundation: configuration
    validation (build-time and request-time — see [Startup &
    Configuration Safety](#startup--configuration-safety)); a logging
    abstraction; the permission engine with both a server-side reader and
    a client-side hook; the real session module; an active-scope provider
    if multi-tenant.
  - `api/base/` — the single low-level HTTP client, callable from Server
    Components, Route Handlers, Server Actions, and Client Component
    query hooks alike.
  - `shared/contracts/` — protocol-level types shared by `api/base` and
    any shared UI that renders server data generically.
  - `design-system/` — the design-token file and the generated theme; the
    theme *provider* lives in `providers/`, since providers must be
    Client Components.
  - `layouts/` — the shell *component* implementations (a main
    authenticated shell, an auth-flow shell), rendered by the matching
    `app/**/layout.tsx` file, which stays a thin wrapper.
  - `shared/` — purely technical, globally reusable, business-agnostic
    building blocks — same catalogue as the base guide.
  - `pages/` — top-level, non-module pages, rendered by a matching
    `app/**/page.tsx`.
  - `modules/` — the business domain layer; see [Module & Feature
    Convention](#module--feature-convention). Each module's `pages/`
    folder holds the real page component (a Server Component by default)
    and exports a small route-metadata object for navigation.
  - `routes/` — no longer assembles a route *tree* — Next.js's file
    system already is the tree. Holds the path registry (including
    dynamic-segment path-builder functions) and the aggregated
    navigation-metadata registry.
  - `styles/` — any global, non-component styling.
  - `test-support/` — shared testing infrastructure: a fetch-mocking
    router and its handlers, shared fixtures.
- `next.config.ts` — build/runtime configuration.

---

## Rendering Strategy — Server vs. Client Components

Next.js's App Router renders every component as a **Server Component by
default**; a component (and everything it imports) only becomes a
**Client Component** — shipped as JavaScript to the browser and hydrated
— once it or an ancestor carries a `"use client"` directive.

This is the single biggest architectural shift from a traditional React
SPA, and it's why several sections of this guide read differently from
the base version:

- **Server Components** can be `async`, read cookies/headers, call a
  database or the base API client directly, and never ship their code (or
  their dependencies) to the browser. They cannot hold state, run
  effects, use browser-only APIs, or attach event handlers.
- **Client Components** behave like the base guide's components: state,
  effects, context, event handlers. They're the only place TanStack
  Query, browser storage, or any interactive library runs.

**Default to Server, opt into Client at the leaf (Rule 27).** A page, a
layout, a section — these stay Server Components unless they have a
concrete reason not to. `"use client"` is added only at the smallest
component that actually needs interactivity, never at a page or layout
root "to be safe." A Client Component's entire subtree ships to the
browser, so marking a large composed section as Client just because one
button inside it needs a click handler silently defeats the model.

**Where this changes layer ownership:**

| Layer | Rendering model |
| --- | --- |
| `app/**/page.tsx`, `app/**/layout.tsx` | Server Component by default |
| `module/pages/` | Server Component by default — fetches its own data directly, or composes Server sections |
| `module/features/*/sections/` | Server Component where possible; Client only if the section itself needs interactivity, not just a descendant of it |
| `module/features/*/components/` | Mixed — a purely presentational card can stay a Server Component; a form input, a dropdown, anything with an event handler is a Client Component |
| `module/features/*/application/` (TanStack Query hooks) | Always Client — hooks require a Client Component |
| `providers/` | Always Client — providers use context |
| `core/permissions/`, `core/session/` | The engine and the data-reading logic are plain TypeScript, usable from either side; the hook (`usePermissions()`) is Client-only, the server helper (e.g. reading the session from cookies) is Server-only — see [Permissions & Conditional UI](#permissions--conditional-ui) |

A component doesn't need to declare anything unless it needs to be a
Client Component — Server is the unmarked default everywhere.

---

## Module & Feature Convention

Business code lives under `modules/`. Each subfolder is a fully isolated
**bounded context** (e.g. `customers`, `billing`, `scheduling`). Inside a
module, split code by concern:

- `modules/<context>/`
  - `api/` — hand-written DTO types, endpoint functions, mapper
    functions, and this module's query-key/cache-tag factory
    (scope-prefixed if the app is multi-tenant).
  - `i18n/` — copy shared across this module's features.
  - `domain/` — pure business logic, framework-free — including this
    module's permission-key vocabulary.
  - `shared/` — code reused across this module's features but not meant
    to leave the module.
  - `features/<feature>/` — one folder per user-facing capability; see
    the feature-level rows of [Layer Responsibilities](#layer-responsibilities)
    and note the [Rendering Strategy](#rendering-strategy--server-vs-client-components)
    table for which sub-layers are Server vs. Client.
  - `pages/` — the module's actual page components, **Server Components
    by default**, imported and rendered by the matching `app/**/page.tsx`.
  - `routes-meta.ts` (or equivalent) — this module's navigation metadata
    (label, icon, order, required permission), consumed by the central
    navigation registry (see [Routing](#routing)) — Next.js's file system
    already defines the actual paths, so this file carries *metadata*
    only, not the route tree itself.
  - `index.ts` — the module's gateway: only the symbols other modules are
    actually meant to consume.

A feature is a self-contained slice inside a module. Two features in the
same module never import from each other directly (Rule 1); anything they
both need bubbles up to `module/shared/` or `module/domain/` (Rule 2) —
unchanged from the base guide.

---

## Scaling Model — From One Module to an Enterprise System

The same shape scales without being restructured, and Next.js's file
system does part of the scaling work for you:

- **One module.** A single bounded context, owning its own path segment
  under `app/`.
- **Several modules, one team.** Cross-module rules start to matter — see
  [Cross-Module Communication](#cross-module-communication).
- **Many modules, many teams.** Each module already owns its own segment
  of the `app/` tree, so the file system stays modular by construction —
  unlike a hand-maintained route config, there's no single growing file to
  split. What still needs deliberate ownership at this scale is
  [Documentation Governance](#documentation-governance--definition-of-done)
  and the navigation-metadata aggregation point.

---

## Cross-Module Communication

Unchanged from the base guide. Pick a rough layering of your own domain —
transactional modules may depend on reference modules, never the reverse
— and enforce that direction. Use one of these sanctioned mechanisms only:

1. **Gateway import** — through the other module's `index.ts`, never its
   internals.
2. **URL navigation** — link to a page in the other module instead of
   importing its component tree directly.
3. **Cache invalidation** — invalidate a shared, well-known query key or
   cache tag across the boundary instead of calling a function directly.
4. **Promotion to `shared/`** — once a piece of code is genuinely needed
   by more than one module, not just anticipated to be.

A circular dependency between two modules is almost always a sign that the
shared piece belongs in `shared/`, or in a new, lower-level module both
can depend on.

---

## Route Grouping

Next.js turns this from a convention into a **first-class routing
feature**: a parenthesized segment inside `app/` — `(dashboard)`,
`(auth)`, `(marketing)` — groups routes and lets them share a layout,
without adding a segment to the actual URL. Unlike the base guide, where a
group folder was purely an organizational convention feeding a manually
declared config, here the group folder **is** the mechanism that attaches
a shared `layout.tsx` to a set of routes.

Why bother:

- Keeps related business areas together, and lets them share a layout,
  without leaking that grouping into URLs.
- Separates dashboard, public, and auth concerns cleanly as the app grows
  — each with its own root `layout.tsx` (its own providers, its own
  shell).
- Makes ownership clearer for teams working on different domains.
- Lets a non-business grouping be marked just as clearly as a real layout
  concern.

Each module's pages typically live under exactly one route group, matching
which shared layout they need (e.g. every authenticated business module
under `(dashboard)`).

---

## Layer Responsibilities

The examples below use a generic `customers` module for illustration.

### Root & Infrastructure Layers

| Folder / Layer | Responsibility | Contains | Forbidden |
| --- | --- | --- | --- |
| `app/` | Next.js's reserved routing root — thin route-segment files only. | `layout.tsx`, `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `template.tsx`, `route.ts` Route Handlers, route groups, dynamic segments. Each file imports and renders a real implementation; sets metadata. | Business logic, data mapping, validation logic — anything that isn't routing/composition glue. |
| `middleware.ts` | Edge-level cross-cutting checks, run before a request reaches a route. | Auth redirect, active-scope cookie check, locale detection. Business-blind, same spirit as `core/` (Rule 16). | Business logic, data fetching, anything domain-specific. |
| `providers/` | The composition-root Client Component tree, mounted once from the root `layout.tsx`. | Global providers (theme, data-fetching client), the top-level client-side error boundary. | Business logic, feature workflows, domain rules, feature state. |
| `bootstrap/` | Startup logic run once per server instance. | Startup logging registration and similar one-time setup. | UI components, layouts, hooks. |
| `core/` | Technical, business-agnostic system foundation. | Config validation (build-time + request-time); logging abstraction; the permission engine with server + client bindings; the real session module; an active-scope provider if multi-tenant. | Styled/visual UI components, business-specific models, feature execution. |
| `api/base/` | Low-level network infrastructure — the only layer that parses the backend's response shape. | The base HTTP client, a typed error hierarchy, a binary-download helper, a params serializer, runtime-aware base-URL resolution, Next cache/revalidate option threading. | Business rules, entity types, JSX, a second overlapping HTTP library. |
| `shared/contracts/` | Neutral protocol types shared by `api/base` and generically server-data-aware shared UI. | A response-envelope type, page/cursor response shapes, a generic filter-definition type. | Endpoint functions, runtime logic. |
| `design-system/` | Application theming layer and design tokens. | The token file, the generated theme. The theme *provider* lives in `providers/`. | Business logic, domain rules, feature state, raw hex/px values in features. |
| `layouts/` | App shell component implementations. | A main authenticated shell (navigation driven by route metadata, a topbar), an auth-flow shell — rendered by the matching `app/**/layout.tsx`. | Data fetching beyond shell needs, route declarations, business rules. |
| `shared/` | Purely technical, globally reusable, business-agnostic items. | Generic UI primitives; generic data-heavy composables; generic hooks; formatters; the shared i18n layer. | Domain-aware items, and anything from `core/`, `modules/`, `app/`, or `routes/` (lint-enforced). |
| `pages/` | Top-level, non-module pages. | Landing/home pages not yet belonging to a feature module, rendered by a matching `app/**/page.tsx`. | Business domain logic — prefer `modules/`. |
| `modules/` | Business domain layer. | Each subfolder, following the Module & Feature Convention. Gateways export only consumed symbols; cross-module dependency direction is downstream only. | Cross-module direct file imports bypassing a module's gateway. |
| `routes/` | The path registry and the aggregated navigation-metadata registry — not a route-tree assembler. | Path constants, dynamic-segment builder functions, the aggregated nav-metadata registry. | Hardcoded path strings anywhere else in the app, business logic. |
| `module/pages/` | Page composition; orchestrates features and evaluates page-level permissions — Server Component by default. | A list page composing a list feature and a create feature, gating both server-side and/or via the permissions hook. | Directly handling client-only concerns it doesn't need. |
| `module/domain/` | Pure, framework-independent business core, including this module's permission vocabulary. | The domain entity, small domain helper functions, permission-key constants. | React components, state managers, API requests. |
| `module/shared/` | Reusable code constrained to this module's features. | A shared status-tag component, a shared option list. Query-key/tag factories live in `api/`. | Infrastructure wrappers, globally generic components. |

### Feature-Level Layers (`modules/*/features/*`)

| Folder / Layer | Responsibility | Contains | Forbidden |
| --- | --- | --- | --- |
| `feature/domain/` | Feature-local domain rules. | A search-filtering function local to this one feature. | React context, global actions. |
| `feature/i18n/` | Micro-copy exclusive to this feature. | Button labels, inline messages, error/retry copy. | Shared global/module strings. |
| `feature/components/` | Granular UI blocks — Server or Client depending on need (see [Rendering Strategy](#rendering-strategy--server-vs-client-components)). | A card (often Server), a form input (Client) — permission-driven visibility comes in as a plain boolean prop. | Network logic, global state sync, permission-hook calls inside a leaf (Rule 11). |
| `feature/sections/` | Larger UI structures assembling components, including distinct loading/error/empty states. | A table section, a creation-modal section. | Complex domain logic. |
| `feature/application/` | Workflow coordinator: orchestrates the data-fetching library, modal state, form submission — always Client. | A list query hook, a creation mutation hook, a workflow hook. | Raw HTML/primitive rendering. |
| `feature/state/` | Feature-specific client state, when genuinely needed — Client. | Local derived UI state managers. | Duplicating the data-fetching library's server cache. |
| `feature/hooks/` | Hooks scoped to this one feature — Client. | A feature-local filter hook. | Globally generic utilities. |
| `feature/utils/` | Stateless helpers. | A function bridging a validation schema into the form layer's validator shape. | State management, browser mutations. |
| `feature/validation/` | Runtime input validation, shared between client and server checks. | This feature's validation schema. | Cross-feature shared schemas. |
| `feature/types/` | Types local to this feature. | Component props types. | Sharing outside the feature without promoting to `module/`. |
| `feature/constants/` | Feature-local constants. | Initial form values, static option lists. | Global system parameters. |
| `feature/tests/` | Tests for this feature. | Unit tests, component tests. | Cross-module test setups. |
| `feature/index.ts` | Public boundary. | Explicit named exports. | Blind wildcard re-exports. |

---

## Architectural Rules

Rules 1–26 are unchanged from the base guide — they're rendering-model
agnostic. Four rules are new (27–30), specific to what Next.js's
Server/Client split and mutation model require.

- **Rule 1 (No Reverse / Cross-Feature Imports):** Features never import
  from sibling features directly. Lint-enforced.
- **Rule 2 (Feature Isolation via Bubbling):** Share code between features
  only by promoting it to `module/shared/` or `module/domain/`, only once
  a second real consumer exists.
- **Rule 3 (Strict Downward Dependency Flow):** The global `shared/`
  layer never imports from `modules/`, `app/`, or `routes/`, nor from the
  session/business-aware parts of `core/`. Lint-enforced.
- **Rule 4 (Domain Purity):** `domain/` stays 100% framework-free: no
  React, no hooks, no query client, no direct API calls.
- **Rule 5 (Thin View Layers):** Pages and top-level components compose
  and orchestrate; they don't embed business rules, data mapping, or
  validation logic directly in JSX — Server or Client alike.
- **Rule 6 (Pragmatic State Placement):**

  | State Type | Recommended Tool |
  | --- | --- |
  | Local UI state | Component state, in a Client Component |
  | Server cache | Server Components for initial/non-interactive data; the data-fetching library for client-driven data |
  | Global app state | A dedicated store, added only when justified |
  | Complex feature state | `feature/state/` |

- **Rule 7 (Server State Source of Truth):** The data-fetching library
  owns all client-fetched server data. Never copy a query result into a
  second state container "for convenience."
- **Rule 8 (Controlled Public Surfaces):** Every module and feature
  exposes an explicit gateway file listing exactly what it re-exports.
- **Rule 9 (Safe Deletability):** Deleting a feature folder must never
  break a sibling; deleting a whole module must never break anything
  outside it.
- **Rule 10 (Encapsulate Declarative Checks):** Route every
  conditional-render permission decision through the engine's helpers —
  see [Permissions & Conditional UI](#permissions--conditional-ui).
- **Rule 11 (Presentation-First UI):** Leaf, purely presentational
  components take booleans and callbacks as props; they never call a
  permission hook, a data hook, or reach into global state themselves.
- **Rule 12 (Module Gateways):** Every module exposes one explicit entry
  file. External layers only ever import through it.
- **Rule 13 (No Architectural Dumping Grounds):** Top-level folders named
  `misc`, `helpers`, `common`, `other`, or `utils` are banned.
- **Rule 14 (DTO Sandbox Boundary):** Raw backend response shapes never
  reach UI components directly — each module owns its own DTO types and
  mappers. Importing another module's DTOs or endpoint functions directly
  is forbidden.
- **Rule 15 (Lightweight Layouts):** A reusable page shell is implemented
  as a Next.js `layout.tsx` and stays a pure wrapper — no data fetching
  beyond what the shell itself genuinely needs — attached once per route
  group, never re-implemented by every page.
- **Rule 16 (Business-Blind Infrastructure):** `core/`, `api/base/`, and
  `middleware.ts` never reference a specific domain concept. Only the
  owning business module assigns meaning to its own permission keys.
- **Rule 17 (Distributed Localization Ownership):** Feature-local copy
  lives in the feature; module-shared copy at the module level; truly
  generic words in the shared i18n layer. See [Appendix
  A](#appendix-a--french-language-naming-convention-french-client-projects-only)
  for French client projects.
- **Rule 18 (Permission-Driven UI Composition):** Evaluate permissions
  once, at the highest boundary that can act on them — see [Permissions &
  Conditional UI](#permissions--conditional-ui) — then pass the result
  down as props or a guard.
- **Rule 19 (Centralized Route Paths):** Never hardcode a path string in
  a `<Link href>`, a `redirect()`, a `router.push()`, or a Server
  Action's return, outside of the single path registry — including the
  functions that build a dynamic segment's URL. A shared not-found/
  access-denied surface linking to the literal root path is a documented,
  deliberate exception (Rule 3).
- **Rule 20 (Response Envelope Boundary):** Whatever shape your backend's
  responses take, parse it in exactly one place — the base API client.
- **Rule 21 (Scoped Context Headers & Keys — if multi-tenant):** Any
  tenant/scope header or cookie is read and injected exclusively through
  the base API client and a small, centrally owned session/scope-reading
  helper — features never call `cookies()`/`headers()` directly, and
  every query key or cache tag embeds the active scope.
- **Rule 22 (Idempotent Critical Creations):** Any creation endpoint for
  a financial or otherwise hard-to-undo resource carries a
  client-generated idempotency key, stable across retries of the same
  submission attempt.
- **Rule 23 (Precise Numeric & Money Handling):** Monetary and other
  precision-sensitive values are treated as exact strings end-to-end —
  never floating-point math client- or server-side, never a client- or
  server-computed total the backend didn't already provide.
- **Rule 24 (Restricted Optimistic Updates):** Optimistic UI updates are
  reserved for low-risk, easily reversible mutations. Anything touching
  inventory, money, or another system of record waits for the server's
  response, then invalidates the narrowest correct keys/tags.
- **Rule 25 (Permission Keys Verbatim From Backend):** Client-side
  permission keys are the backend's own catalog, imported verbatim — the
  server (or a server-side check) re-checks every permission on every
  request; hiding a control client-side is UX, never a security boundary
  on its own.
- **Rule 26 (Design Tokens as Single Source of Truth):** Colors, spacing,
  control heights, radii, shadows, and typography come from the
  design-token layer — never hardcoded as raw values in feature code.
- **Rule 27 (Server-First Component Boundary):** Components are Server
  Components by default; `"use client"` is added only at the smallest
  leaf that genuinely needs state, effects, or browser APIs — never at a
  page or layout root "just in case." See [Rendering
  Strategy](#rendering-strategy--server-vs-client-components).
- **Rule 28 (Environment Exposure Boundary):** Only variables explicitly
  prefixed for client exposure are ever readable from a Client Component.
  Every other value — API tokens, internal service URLs, signing secrets
  — stays server-only, validated in the same central config module, and
  is never imported into a file carrying `"use client"` (lint-enforced).
- **Rule 29 (One Path for Mutations):** A Client Component never calls a
  third-party or backend API directly from the browser. Every mutation
  goes through a Server Action or a Route Handler — the one place
  validation, auth/permission checks, and revalidation for that mutation
  actually happen. Mirrors Rule 20's single-parsing-layer principle,
  applied to the write path.
- **Rule 30 (Explicit Revalidation Ownership):** Each data-fetching
  function declares its own caching/revalidation policy — a time-based
  window, or on-demand tag/path revalidation triggered by the mutation
  that changes it — at the point closest to the data. A mutation that
  changes data invalidates the same tag its read side was tagged with.

---

## Startup & Configuration Safety

Next.js gives configuration validation two different moments to fail in,
and the safe pattern uses both deliberately instead of collapsing them
into one:

- **Build-time / server-startup validation** — required variables the
  server needs to even start (a database URL, a required API base URL)
  are validated once, at server startup, by a pure function that returns
  a discriminated result and never throws mid-import. If validation
  fails, failing the build or refusing to start the server **loudly** is
  the correct behavior — unlike a SPA, the deploy hasn't reached any user
  yet, so a loud failure here is strictly safer than a quiet one at
  request time.
- **Request-time / client-exposed validation** — values only known per
  request (the active scope, a feature flag) or values read on the client
  (the client-exposed subset — see Rule 28) are validated the same way,
  but a failure here can't halt the whole server for every other request.
  It should produce a typed error the nearest boundary renders.

**Rendering the failure:** Next.js's own per-segment `error.tsx` boundary
(and the root `global-error.tsx` for a failure in the root layout itself)
replaces a hand-built startup error screen — a thrown error in a Server
Component during render is caught by the nearest one automatically. Keep
`global-error.tsx` dependency-free (no theme, no design tokens, no i18n)
for the same reason a fallback screen always is: it must be able to
render even when something foundational is broken.

**Rule of thumb:** nothing in `core/` should throw synchronously at
module *import* time — that failure mode still produces an unrecoverable
response. A thrown error *during render*, inside a Server or Client
Component, is fine — Next.js's boundaries exist precisely to catch that.

---

## HTTP & Data Fetching

Keep exactly one low-level HTTP layer — a small, envelope-aware wrapper
around `fetch`. In a Next.js app it's called from more places than in a
SPA, and needs two small extensions:

- **Runtime-aware base URL.** A Server Component or Route Handler may
  need an internal service address while a Client Component must call a
  public one. Resolve the base URL once, based on which runtime is
  calling, inside the base client — never hardcoded per call site.
- **Cache/revalidation options threaded through.** Next.js extends
  `fetch` with its own caching directives (a time-based revalidation
  window, or tags for on-demand invalidation). The base client accepts
  these as an explicit, typed option rather than every call site reaching
  past the wrapper to set them directly.

**Where data actually gets fetched now has a real choice to make:**

1. **Server Components fetch directly.** For anything that doesn't need
   client-driven refetching, pagination, or optimistic updates, an
   `async` Server Component calls the base client (or a module's `api/`
   function built on it) directly, and the data is already in the initial
   HTML — no client-side loading spinner, no second network round trip.
2. **Client Components use the data-fetching library.** Once a piece of
   UI needs to refetch on a client-driven interaction (a filter changing,
   a poll, an optimistic mutation), it becomes a Client Component and
   uses TanStack Query, built on the same base client and the same
   module-owned key/tag factory.

**The pattern for any new endpoint** is the same three layers as always —
a protocol-only function in the module's `api/`, a mapper that converts
DTOs into domain entities and normalizes errors, and a consuming layer on
top — except that consuming layer is now either a Server Component
calling it directly, or a Client Component's query hook, depending on
where the data belongs per the rule above.

**Error states are still not optional.** A Server-Component-rendered list
that fails to fetch renders its error state directly, or lets the nearest
`error.tsx` catch it; a Client-Component-rendered list still branches on
loading → error → empty → data with a retry action. Collapsing "the
request failed" into "there's no data" is exactly as real a bug here as
anywhere else.

---

## Data Layer at Scale

Conventions that keep hundreds of queries and fetches coherent as modules
accumulate — whichever side of the Server/Client boundary they're on.

### Query keys & cache tags: one factory per module

Client-side query keys are structured exactly as in a plain SPA:
hierarchical, never ad hoc, one factory per module. Server-side `fetch`
calls that need on-demand revalidation use the equivalent idea — a
**cache tag**, generated by the same per-module factory, so a Server
Action that mutates a resource can revalidate precisely the tag its read
side used, instead of a broad, unscoped revalidation.

If the app is multi-tenant, both the query keys and the cache tags are
prefixed with the active scope (Rule 21), so a scope switch can never leak
one tenant's cached data — client or server-cached — into another's view.

### Staleness & revalidation tiers

| Data kind | Example | Client `staleTime` | Server `revalidate` | Rationale |
| --- | --- | --- | --- | --- |
| Reference / config | payment terms, tax rates, user preferences | 5–15 min | 5–15 min, or on-demand tag revalidation on change | Rarely changes mid-session; refetching on every request is waste. |
| Transactional lists | invoices, orders, stock | 30–60 s | On-demand tag revalidation from the mutation that changes them | A mutation should invalidate its own tag rather than waiting out a timer. |
| Detail being actively edited | an open record | Always fresh, plus targeted invalidation | Never cached | Editing demands freshness. |

### Pagination & big lists

Server-side pagination by default beyond a few hundred rows; cursor-based
infinite loading reserved for genuinely append-only feeds with an
explicit "load more" action. A first, non-interactive page of a list is a
good candidate for a direct Server Component fetch; the moment a list
needs client-driven filtering, sorting, or pagination without a full page
reload, it becomes a Client Component backed by the data-fetching
library.

### Invalidation etiquette

The narrowest correct key or tag only, triggered by the mutation that
actually changed the data — a Server Action calling its tag-revalidation,
or a Client Component's success handler — never a broad, unscoped
revalidation of the whole app. Optimistic updates are restricted per
Rule 24.

---

## Forms & Tables at Enterprise Scale

### Forms

Two legitimate paths, chosen deliberately per form rather than by habit:

- **Server Actions, by default.** A form posts to a Server Action, which
  validates with the same schema the client used, performs the mutation
  through the base API client, and triggers the appropriate tag/path
  revalidation (Rule 30) before returning. Pending and error state come
  from the framework's own form-status hooks rather than a hand-managed
  client mutation hook. This is the right default for straightforward
  create/update/delete forms that don't need optimistic UI or to stay
  open through a multi-step client-managed flow.
- **A Client Component + a mutation hook**, for forms that genuinely need
  richer client-side interactivity — optimistic UI, inline validation
  tied to other client state, or embedded in a larger client-managed
  workflow (a multi-step wizard, a modal stack).

Whichever path is chosen:

- One validation schema per form, shared between client-side validation
  and the server-side check that actually enforces it — client validation
  is UX only, never the security boundary (same spirit as Rule 25).
- Long forms (roughly 8+ fields, or more than one logical group) render as
  a stepper or tabs of section components, validating independently
  against a subset of the same schema.
- Server-side field errors map onto form fields through the mapper layer,
  normalized into one consistent shape regardless of whether they came
  back from a Server Action or a Route Handler.
- Money and quantity inputs preserve their exact string value (Rule 23).
  Destructive or terminal actions go through a confirmation dialog, with
  a required justification input when the domain calls for an override.
- Creations on idempotent resources attach their key via a shared helper
  — never hand-managed per feature.

### Tables (grids)

Backend-provided filter metadata where available, server-side
filtering/sorting/pagination beyond a few hundred rows, URL-driven filter
state, feature-local row selection, virtualization past roughly 20
columns or 100 unpaginated rows, and the same four states everywhere. One
addition: a Server Component can read the initial `searchParams` directly
and render the first filtered page with zero client JS — a Client
Component only takes over once the table needs to change its own filters
without a full navigation.

---

## Error-Handling Taxonomy

Every failure still falls into exactly one bucket, but several now map
onto a native Next.js primitive instead of a hand-built one:

| Kind | Detected where | Rendered how | Never |
| --- | --- | --- | --- |
| Network / timeout | Base API client | Section-level error state + retry (Client), or the segment's `error.tsx` (Server) | A toast that vanishes while the page looks empty |
| Backend validation (4xx payload) | Server Action / mutation error handler | Field-level errors mapped onto the form | A generic "something went wrong" for a recoverable case |
| Permission denied | `middleware.ts` / a Server Component's own check / the permissions hook | A redirect, a `notFound()`, an access-denied page, or a hidden control — see note below | Showing disabled controls the user can never enable |
| Unexpected (bug, 5xx, thrown render error) | The nearest `error.tsx`; `global-error.tsx` for a root-layout failure | Boundary fallback with a report action; logged | Swallowing into the console only |
| Environment / config | Startup validation | Build failure (build-time) or `global-error.tsx` (request-time) | Anything else rendering first |

**A deliberate choice for permission denied:** many enterprise apps
intentionally resolve an unauthorized resource to `notFound()` rather
than a distinguishable "access denied" page, so an unauthorized user
can't tell "doesn't exist" from "exists, but not for you." Decide this
once, document it, and apply it consistently.

**Rule of thumb:** errors a user can act on render inline where the
action is; errors they can't act on escalate to a boundary. Toasts are
reserved for success confirmations and background-action failures, never
for primary view loading failures.

---

## Permissions & Conditional UI

Same three-tier model as any React architecture — engine, bindings,
vocabulary — with one real upgrade: Next.js lets a permission check
happen **before any HTML is sent**, not just after the client has already
loaded the page.

1. **The engine** — pure functions comparing a permission string against
   a list of permission strings, zero business knowledge.
2. **The bindings** — now three, not one:
   - `middleware.ts` — a coarse, section-wide check (is this user
     authenticated at all; do they have access to this whole route
     group) that can redirect before the route even starts rendering.
   - A **server-side check inside the page/layout itself** — reads the
     session server-side, and calls `redirect()` or `notFound()` before
     fetching or rendering anything the user isn't entitled to see.
     Strictly more secure than a client-only gate, since unauthorized
     data is never even serialized to the client.
   - The **client hook + guard component** — for conditionally rendering
     a piece of UI *within* an already-authorized page.
3. **The vocabulary** — each module's own permission-key constants; only
   the owning module knows what a key means (Rule 25).

**When to use which:**

| Situation | Use |
| --- | --- |
| Nobody without a given permission should ever receive the page's HTML at all | A server-side check in the page/layout, returning `redirect()`/`notFound()` |
| An entire route group needs the same coarse check | `middleware.ts` |
| Toggling a chunk of JSX within an already-authorized page | The guard component |
| Need the boolean to gate a client-side query or combine with other client state | The permissions hook |
| A leaf, purely presentational component needs to vary by permission | A plain boolean prop passed down from whichever tier above already evaluated it |

The server always re-checks (Rule 25) — client-side gating remains a UX
convenience layered on top of a real server-side boundary, never a
replacement for one.

---

## Routing

Next.js's App Router derives the entire route tree from the `app/` file
system — there's no separate config file declaring routes as data. Each
route segment is a folder; the files inside it take on fixed meanings:

- `page.tsx` — the route's content (thin — imports and renders a Server
  Component from `modules/<context>/pages/`).
- `layout.tsx` — a shell wrapping this segment and everything nested
  under it (thin — imports and renders a component from `layouts/`).
- `loading.tsx` — an instant loading UI shown while the segment's data
  resolves.
- `error.tsx` — the segment's own error boundary.
- `not-found.tsx` — this segment's 404 state.
- `route.ts` — a Route Handler, for cases outside a page's own data needs.
- Dynamic segments (`[id]`), catch-alls (`[...slug]`), and route groups
  (`(group)`) shape the URL without extra configuration.

**What replaces "one declaration, two consumers."** A hand-maintained SPA
route config can derive both the route tree and the nav menu from one
file. Next.js's file system already gives you the route tree for free,
but not a nav menu — so each module still exports a small route-metadata
object (label, icon, order, required permission) for its own top-level
pages, and a central navigation-metadata registry in `routes/` aggregates
these to drive the sidebar. One declaration per module, imported by
exactly one aggregator.

**Route guards** happen in two places — see [Permissions & Conditional
UI](#permissions--conditional-ui): `middleware.ts` for a coarse,
whole-route-group check, and a server-side check inside the page/layout
for anything fine-grained or data-dependent.

### Route Tree at Enterprise Scale

Next.js handles this well by construction: each module simply owns its
own path segment inside `app/` (e.g. every `customers`-module page lives
under `app/(dashboard)/customers/`), so the file tree itself stays
modular without anyone needing to manually split a growing config file.
The central navigation-metadata registry stays the only place the whole
app's *navigable menu* is visible.

**Lazy-loading is automatic.** Next.js code-splits by route segment on
its own; there's no manual lazy-import wiring to maintain. The discipline
that still matters is Rule 27: a segment marked `"use client"`
unnecessarily still ships its whole subtree's JavaScript to the browser,
regardless of route-level splitting.

---

## Centralized Route Paths

One single registry is still the only source of literal route path
strings — including the **functions that build a dynamic segment's URL**
(a customer-detail path built from an id), since those are exactly the
strings most likely to be duplicated slightly differently across the app
otherwise. Every `<Link href>`, `redirect()`, and `router.push()` call
references the registry instead of constructing a path inline. A shared
not-found/access-denied surface linking to the literal root path remains
a documented, deliberate exception (Rule 19).

---

## Multi-Tenancy & Active Scope (Optional)

*Skip this section entirely if the application is single-tenant.*

Same model as any multi-tenant React app, with one adjustment: since the
active scope needs to be readable both server-side (Server Components,
Route Handlers, `middleware.ts` — all via the framework's cookie-reading
API) and client-side (a Context provider in `providers/`), its source of
truth is a **cookie** rather than client-only storage, with the client
Context hydrated from that same cookie.

1. Bootstrap it from a single "who am I / what do I have access to"
   endpoint, persist the choice in the cookie, and revalidate it
   server-side (in `middleware.ts` or the root layout) on requests that
   matter.
2. On switch: validate the new scope against the backend, update the
   cookie, cancel/remove client-cached queries that aren't explicitly
   exempted, and revalidate any server-cached tags scoped to the previous
   value (Rule 21) — never a blanket cache clear on either side.
3. Handle two edge cases explicitly: no accessible scope at all, and the
   active scope being revoked mid-session — both are naturally easier to
   enforce here, since `middleware.ts` can catch them before a page even
   starts rendering.

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
Each action is typically a **Server Action** (Rule 29), which re-validates
the transition server-side regardless of what the UI already disabled.

- Terminal statuses render read-only with a short explanation of why.
- Cancellation and reversal actions get honest copy about what will
  actually happen — a status flip for something still in draft is not the
  same operation as a compensating action for something already committed
  downstream.
- Any override of a normal transition rule requires an explicit
  justification input, never a silent flag.
- Illegal transitions are ultimately refused by the backend — surface
  that refusal inline at the action, then refresh the entity's actual
  state rather than trusting an optimistic guess.

---

## Design Tokens & Theming

Same principle as any React app — one single source of visual truth
regardless of UI library. Next.js adds two practical notes: the generated
theme's *provider* is necessarily a Client Component (context requires
it), mounted once via `providers/` rather than scattered per page; and a
CSS-variable-based token file (paired with plain CSS or a utility-CSS
framework) works especially well here, since it needs no Client Component
at all to apply — only the parts of the design system with actual
interactive behavior (a theme toggle, a component using a styling
library's runtime context) need to be Client Components.

---

## Backend Integration Contract

Same documentation discipline as any enterprise frontend. One decision
worth making explicit up front: is Next.js a **thin frontend to a
separate backend service** (Server Actions and Route Handlers act as
authenticated proxies, keeping API tokens server-only — a genuine
security upgrade over a SPA, where such tokens had to either live in the
browser or go through a separate BFF), or is Next.js **the backend
itself** (Route Handlers and Server Actions call a database or internal
service directly)? The rest of this guide assumes the former, for
continuity with the wider architecture, but every principle below applies
equally to the latter — just replace "the backend" with "this app's own
data-access layer."

At minimum, document:

- **Response envelope** — does every response share a common wrapper? Are
  any endpoints exempt (binary downloads, a health check)?
- **Pagination strategy** — offset/page-based by default, with any
  exceptions for append-only, ledger-style endpoints using cursors.
- **Filtering & sorting** — ideally backend-provided metadata per
  resource, so the frontend never hardcodes filter enums.
- **Cross-cutting headers** — auth token, tenant/scope headers, an
  idempotency-key convention, a CSRF-token echo — confirmed to live
  exclusively in the base API client (Rule 20).
- **Error-code taxonomy** — stable, machine-readable codes the frontend
  can safely branch on, distinct from a free-text display message.
- **Lifecycle transition rules**, if entities have one — where they live,
  and what error code an illegal transition returns.

Keep a short "contract drift" log of any backend behavior the frontend
has learned about that isn't yet reflected in the official API docs.

---

## Performance Budgets

Budgets are enforced by review and CI, not vibes — with Next.js shifting
part of the conversation from raw bundle size to **how much of the tree
opted into `"use client"` unnecessarily** (Rule 27), since Server
Components ship no JavaScript to the browser by default.

- **Client JS payload** — scoped to whatever's actually inside
  `"use client"` boundaries; a route that looks server-rendered but drags
  in a large client subtree because one layout was marked client-side
  unnecessarily is exactly the regression to catch.
- **Core Web Vitals** (LCP, INP, CLS) are the real, user-facing target —
  track them in production, not just a synthetic bundle number.
- **Images and fonts** — the framework's built-in image and font
  optimization are the enforced default over a hand-rolled `<img>` tag or
  a manually linked web font; both are low-effort, high-impact wins.
- **No module is ever eagerly imported** into a Server Component that
  doesn't need it, and no Client Component pulls in a heavy dependency
  that could instead stay server-side.
- **Lists virtualize or paginate** beyond roughly 100 rendered rows.
- **Regression ritual** — compare the production build's per-route
  segment size against the previous release before merging a module, and
  investigate any significant growth; automate it as a CI gate.

---

## Naming Conventions & Code Style

Unchanged from the base guide — this is a language-level convention, not
a framework-level one. All application-created names are **English by
default**. See [Appendix
A](#appendix-a--french-language-naming-convention-french-client-projects-only)
for the French-language variant used only on French client projects.

### General Rules

- Prefer descriptive names over abbreviations.
- Keep well-known ecosystem abbreviations as-is: `API`, `UI`, `URL`, `ID`,
  `DTO`.
- Never reuse the same word for two different concepts.
- Avoid vague catch-all names: `data`, `stuff`, `temp`, `thing`, `helper`,
  `common`.
- Next.js's own reserved filenames (`page.tsx`, `layout.tsx`,
  `loading.tsx`, `error.tsx`, `route.ts`, `middleware.ts`, and so on) are
  framework-required identifiers, left unchanged, exactly like
  `ThemeProvider` or `next.config.ts` — the same "framework-required
  identifiers stay unchanged" exception as the Language Policy in
  Appendix A.

### Folder Naming Rules

- Ordinary folders use `kebab-case` — `customer-list`, `create-customer`.
- Group folders use parentheses: `(dashboard)`, `(marketing)` — and in
  this project are literal Next.js route groups, not just an
  organizational convention (see [Route Grouping](#route-grouping)).
- Feature and module names describe a bounded context: `customers`,
  `billing`, `scheduling` — never `misc` or `other`.
- Every module exposes an index/gateway file.

### File Naming Rules

- **Components:** `PascalCase.tsx` — `CustomerCard.tsx`,
  `CustomerForm.tsx`, `PermissionGuard.tsx`.
- **Hooks:** `use` + PascalCase — `useCustomerFilters.ts`,
  `useCreateCustomer.ts`, `usePermissions.ts`.
- **Server Actions:** named after the mutation they perform, same verb
  vocabulary as mutation functions — `createCustomer.ts` exporting a
  `"use server"` action.
- **Mappers / data-access functions:** named after the action they
  perform — `mapCustomerDtoToCustomer.ts`-style logic inside
  `customerMapper.ts`, `listCustomers.ts`.
- **Test files:** subject-first, matching the project's standard test
  suffix — `CustomerTableSection.test.tsx`, `permissionEngine.test.ts`.
  Test *description strings* read as documentation of behavior for any
  engineer, regardless of the project's identifier-naming-language
  policy.
- **Infrastructure code without a "feature" skeleton** colocates its test
  next to the source file rather than using a `tests/` subfolder.

### Class and Type Naming Rules

- `PascalCase`, no `I`/`T` prefixes.
- Components named for what they render: `CustomerCard`, `CustomerForm`.
- Domain entities named after the business concept: `Customer`.
- DTOs suffixed `Dto`: `CustomerDto`, `CreateCustomerRequestDto`.
- Frontend contract types follow one consistent pattern:
  `<Concept>Response` / `<Action><Concept>Request`.
- Permission-related types split into an opaque string alias at the
  engine level and a module-owned literal union at the vocabulary level.

### Method and Function Naming Rules

- **Queries:** `get`, `list`, `find` — `listCustomers`.
- **Mutations:** `create`, `update`, `delete` — `createCustomer`.
- **Transformations:** `map`, `build`, `format`, `filter`, `validate`.
- **Boolean checks:** `is`, `has`, `can`, `should`.
- **Prop callbacks:** the framework's own convention — `onSubmit`,
  `onCancel`, `onRetry`.

### Variable Naming Rules

- `camelCase`.
- Booleans read as a state or a question: `isLoading`, `canEdit`,
  `hasErrors`, `isModalOpen`.
- Collections are plural nouns: `customers`, `filteredCustomers`.
- Identifiers keep the recognized `Id` suffix: `customerId`.
- Timestamps follow one consistent convention applied everywhere —
  `createdAt`, `updatedAt`.

### Enum and Constant Naming Rules

- Constants: `UPPER_SNAKE_CASE` — `DEFAULT_PAGE_SIZE`,
  `SEARCH_DEBOUNCE_MS`.
- Permission keys are the backend's own catalog codes, used verbatim,
  grouped under one constant object per module — never invented
  independently on the frontend.
- Route paths follow whatever casing the URLs are actually served in,
  collected in the single path registry.
- Contractual enum values (a literal string the backend sends) stay
  verbatim; the human-readable label lives in a separate, module-owned
  mapping.

### i18n Key Naming Rules

- Translation objects use camelCase keys.
- Keep keys scoped to the feature or module that owns the copy, unless a
  string is genuinely reused everywhere (Rule 17).

### Naming Anti-Patterns to Avoid

- Vague filler words as variable names.
- Dumping-ground folder names at the top level.
- Mixing two naming languages in the same file (relevant mainly on
  French client projects — see Appendix A).
- Copying backend field names straight into UI code without going
  through the mapper layer.

### Naming Priority Order

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
| `core/`, `shared/`, `api/base/`, `providers/`, `routes/` | Colocated — same directory as the source | Infrastructure is stable, few files per folder. |
| `modules/<m>/api/` and `modules/<m>/domain/` | Colocated — same directory as the source | Module-level infrastructure follows the same rationale. |
| `modules/<m>/features/<f>/` (any sub-layer) | `feature/tests/` subfolder | Features grow to many files across sub-layers — a dedicated tests folder prevents clutter. |

**Decision shortcut:** infrastructure for the module → colocate. Part of
a feature → `features/<f>/tests/`.

### Additional rules

**Server Components are async and can't be rendered directly with a
component-testing library** the way Client Components can — this is the
one structural change from a plain SPA's testing story.

- **Server Components:** test the logic, not the rendered tree — extract
  data-fetching, mapping, and business rules into the same testable
  `domain/`/`api/` functions this guide already isolates them into
  (Rule 4), and unit-test those directly. Cover the actual rendered
  output of a Server Component through end-to-end tests instead.
- **Client Components:** unit/component-tested with a
  behavior-focused testing library, same as any React app.
- **Server Actions and Route Handlers:** plain async functions — call
  them directly in a test, mocking the underlying service call.
- **End-to-end coverage carries relatively more weight** here than in a
  plain SPA, since it's the most direct way to verify a full route
  segment — Server and Client parts together — actually renders
  correctly. Cover the flows that would be genuinely bad to ship broken;
  run on a schedule and before releases if they're slow.
- Test description strings read in English, regardless of the project's
  identifier-naming-language policy (see Appendix A).
- CI runs lint, typecheck, test-with-coverage, and build on every push
  and pull request.
- Prefer a lightweight, dependency-free approach to mocking network calls
  at the fetch level, usable from both Server- and Client-side tests,
  over a heavier mocking framework.

### Ownership matrix (enterprise scale)

| Layer | Owner | Required tests | Category |
| --- | --- | --- | --- |
| `domain/` (pure logic) | Owning module team | Exhaustive unit tests | Unit |
| `api/` mappers & protocol functions | Owning module team | Unit tests, mocked network | Unit |
| Server Actions / Route Handlers | Owning module team | Direct function-call tests, mocked service layer | Unit/integration |
| `application/` hooks (Client) | Owning module team | Hook behavior: states, invalidation, optimistic rollback | Unit/integration |
| `sections/`, `components/` (Client) | Owning module team | Render states incl. loading/error/empty | Component |
| Full route segments (Server + Client together) | Owning module team | Smoke coverage of the actual rendered page | End-to-end |
| `core/`, `shared/`, `api/base/` | Platform owner | Colocated unit tests, high bar | Unit |
| `middleware.ts`, route wiring | Platform owner | Type-checked by the file-system convention itself; dedicated tests once guards get complex | — |

A module isn't "done" until its test pyramid is green and its `domain/`
and `application/` layers show meaningful coverage; a numeric
UI-coverage target is deliberately not set.

---

## Tooling & Automation

| Tool | Enforces |
| --- | --- |
| Linter custom rules/overrides (layered on Next's own lint configuration) | No feature-to-feature imports (Rule 1); `shared/` dependency direction (Rule 3); a module's internal DTO/endpoint layer never leaks outside it (Rule 14); no dumping-ground folders (Rule 13); no server-only import inside a `"use client"` file (Rule 28). |
| The config-validation module | Validates environment variables, split across build-time and request-time (see [Startup & Configuration Safety](#startup--configuration-safety)). |
| Pre-commit hook + staged-files runner | Runs lint and format only on staged files before every commit. |
| Commit-message hook + convention checker | Enforces a consistent commit-message convention, in English. |
| CI pipeline | Lint, typecheck, test-with-coverage, build — on every push/PR, plus a build-output route-segment size comparison. |
| Fetch-mock harness | A zero-dependency fetch router for all network tests, usable from both Server- and Client-side tests. |
| Automated dependency updates | Scheduled, grouped update PRs — reviewed, not auto-merged for anything touching the runtime. |

---

## Documentation Governance & Definition of Done

### Keeping this document true

- Structural changes to the architecture go through a short decision
  record (context / decision / consequences, one page) rather than
  silently drifting from what this guide says.
- Assign ownership of the platform layers (`core/`, `shared/`,
  `api/base/`, `routes/`, `middleware.ts`) to a named owner or small group
  once more than one team touches the codebase; let each business module
  belong to its owning team.
- This document describes *invariants*; a separate contributing guide can
  carry the how-to. When the two disagree, this document wins until the
  contributing guide is fixed to match.

### Definition of Done for a new module

- [ ] Follows the Module & Feature Convention; the gateway file exports
      only consumed symbols.
- [ ] Owns its own path segment under `app/`; `page.tsx`/`layout.tsx`
      files stay thin; navigation metadata exported and aggregated.
- [ ] Query key / cache-tag factory created; staleness/revalidation
      tiers chosen deliberately; mutations invalidate narrowly.
- [ ] Permission keys defined in the module's own domain layer; `core/`
      untouched. Server-side access checks in place, not just client-side
      gating.
- [ ] `"use client"` only appears at genuinely interactive leaves — not
      at a page or layout root (Rule 27).
- [ ] List/detail views implement all four states; the error-handling
      taxonomy is respected.
- [ ] Mutations go through a Server Action or Route Handler, never a
      direct client-side call to a backend (Rule 29).
- [ ] Standard lint, typecheck, test, and build scripts all pass.
- [ ] Bundle/route-segment impact checked against the performance
      budgets.
- [ ] Any cross-module dependency uses a sanctioned mechanism, or is
      explained by a decision record.

---

## Dead Code Hygiene

Unchanged from the base guide. A generic-sounding constant, type, or copy
string with **no current consumer** is dead code, not a head start — it
costs a future reader time figuring out whether it's safe to touch, and
its presence implies a usage that doesn't actually exist anywhere.

**Rule of thumb:** if you're adding something "because a future feature
will probably need it," it belongs in that future feature's own branch
when it actually arrives — not preemptively promoted into `shared/` or
`module/domain/` today. Promote code upward only once a second real
consumer appears (Rule 2).

---

## Architecture Self-Check

Run through this after any non-trivial change — before opening a PR, not
just once at the end of a big feature.

### Naming & language
- [ ] Every new component/function/variable/file/folder name follows the
      project's chosen identifier language (English by default; French
      only per Appendix A on French client projects).
- [ ] Every new comment, and every doc file touched, reads in English —
      this stays true even on a French client project.
- [ ] Every new commit message follows the team's convention, in English.
- [ ] Every new end-user-facing string matches the application's actual
      target language(s).

### Rendering boundary
- [ ] No `"use client"` at a page or layout root "just in case" — it's
      pushed down to the smallest leaf that genuinely needs it (Rule 27).
- [ ] No server-only value (a token, an internal URL, a secret) is
      imported into a file carrying `"use client"` (Rule 28).
- [ ] Data that doesn't need client-driven refetching is fetched in a
      Server Component, not fetched client-side after mount "just
      because."
- [ ] A component that only needs to be a Client Component for one small
      piece of interactivity hasn't dragged its whole surrounding section
      along with it.

### Layering & imports
- [ ] Nothing in `shared/` imports from `modules/`, `app/`, `routes/`, or
      `core/` (run the linter — this is enforced, not just documented).
- [ ] No feature imports another feature directly — only via
      `module/shared/` or `module/domain/`.
- [ ] A module's internal DTO/endpoint layer is only imported from inside
      that module itself.
- [ ] Nothing in `core/` or `middleware.ts` references a specific
      business concept.
- [ ] Every module/feature touched still has an accurate, narrow gateway
      file.
- [ ] Any module-to-module dependency uses a sanctioned mechanism —
      downstream direction only.

### State & data
- [ ] Server data fetched client-side is only ever held by the
      data-fetching library — no component-state or store copy of a
      query result "for convenience."
- [ ] Any new query/mutation uses the single base HTTP client.
- [ ] Any new list/detail view has three distinct states — loading, error
      (with a retry action), and successfully empty — not two.
- [ ] Any code that reads the raw environment directly, instead of going
      through the config-validation module, is a regression.
- [ ] New queries use the owning module's key/tag factory; mutations
      invalidate the narrowest correct keys/tags; the staleness/
      revalidation tier is chosen deliberately (Rule 30).

### Permissions
- [ ] Anything that must never reach an unauthorized user's browser at
      all is checked server-side (page/layout or `middleware.ts`), not
      just gated client-side.
- [ ] Any client-side conditional render uses the permissions hook (for
      the boolean) or the guard component (for a JSX subtree) — not an
      inline role/permission comparison in JSX.
- [ ] The permission's key is defined in the owning module's domain
      layer, not hardcoded inline.
- [ ] Permission checks happen at the highest boundary that can act on
      them, threaded down as props to any purely presentational
      component that needs them.

### Routing
- [ ] The new route lives in the correct module's segment under `app/`;
      `page.tsx`/`layout.tsx` stay thin.
- [ ] Every navigation call references the path registry (including
      dynamic-segment builder functions) rather than a literal string.
- [ ] Navigation metadata is exported from the module and picked up by
      the central registry — not hand-added to a parallel menu array.
- [ ] A route needing a coarse, whole-section guard has it in
      `middleware.ts`; anything fine-grained is checked in the
      page/layout itself.

### Mutations
- [ ] Every mutation goes through a Server Action or a Route Handler —
      no Client Component calling a backend or third-party API directly
      (Rule 29).
- [ ] The Server Action/Route Handler re-validates with the same schema
      the client used — client validation is UX only.
- [ ] The mutation triggers the correct tag/path revalidation for
      exactly the data it changed (Rule 30).
- [ ] Optimistic updates are limited to the benign-mutation cases in
      Rule 24.

### Backend contract & scope
- [ ] The response envelope is parsed only in the base API client;
      features branch on typed error codes, never raw response bodies.
- [ ] No manual attachment of cross-cutting headers/cookies outside the
      base client and its centrally registered helpers.
- [ ] Query keys/cache tags embed the active scope when the app is
      multi-tenant; no bare full-cache-clear anywhere.
- [ ] List filters render from backend-provided metadata where available.
- [ ] Mandated creations on financial/critical resources carry an
      auto-generated idempotency key.
- [ ] Money/quantity fields use string-preserving inputs; zero
      client- or server-side float math on amounts.
- [ ] Entity/document actions derive from the owning family's status map;
      terminal statuses are read-only; any forced override carries a
      required justification.
- [ ] No hardcoded colors/padding/radii in features — theme props or the
      token object only.

---

## Appendix A — French-Language Naming Convention (French Client Projects Only)

> **This appendix applies only when the delivered product is for
> French-speaking end users, and the team has additionally chosen to
> reflect that by writing business-domain *code identifiers* in French —
> not just user-facing text.** If that isn't this project, **delete this
> appendix**. Nothing else in this guide references it, and every example
> elsewhere in this guide is already in English. Next.js's own reserved
> filenames (`page.tsx`, `layout.tsx`, `route.ts`, `middleware.ts`, …)
> fall under the "framework-required identifiers, unchanged" row below
> regardless of this policy.

### Language Policy

| What | Language | Why |
| --- | --- | --- |
| Code identifiers — components, functions, variables, hooks, types, files, folders, query keys/cache tags, test *subjects* | **French** | Business-domain vocabulary matches how the client and end users actually talk about their own business. |
| Code comments, doc comments | **English** | Comments explain *why* to any engineer who joins the team, regardless of their proficiency in French. |
| Markdown documentation | **English** | Documentation is a developer-facing artifact, read by any engineer on any future project. |
| Git commit messages, PR titles/descriptions | **English** | Keeps project history searchable and reviewable for any contributor. |
| Test *description strings* | **English** | They function as documentation of behavior, read by any engineer and surfaced in CI logs. |
| End-user-facing text (UI labels, buttons, validation messages, emails) | **French** | The application is used by French end users — the product itself must speak their language. |
| Framework- or library-required identifiers (`page.tsx`, `layout.tsx`, `route.ts`, `middleware.ts`, a provider component's required name) | **Unchanged** | These are contractual names owned by the framework or library, not by this codebase. |

**In short:** open any source file and both the *symbols* and the *prose
around them* (comments, docs, commit messages, test descriptions) read in
English. Only end-user-facing text and the business-domain vocabulary of
the code itself are French.

### Technical Terms Glossary

The following universal technical terms stay in **English** even inside a
French-named codebase:

`API` · `UI` · `URL` · `ID` · `DTO` · component · hook · query · mutation
· cache · store · module · feature · layout · page · context · schema ·
field · state · props · type · error · boundary · layer · wrapper ·
config · import · export · test · coverage · CI · commit · Git ·
TypeScript · React · Next.js · Server Component · Client Component ·
Server Action · middleware · fetch · chunk · bundle · memo · Promise ·
async · provider · token · path · routes · permission · guard · toast

### Folder Naming Rules

- Ordinary folders use `kebab-case`, in French — `liste-clients`,
  `creer-client`.
- Group folders use parentheses regardless of language: `(exemple)`,
  `(tableau-de-bord)`.
- Feature and module names describe a bounded context in French:
  `utilisateurs`, `facturation`, `rendez-vous`.

### File Naming Rules

- **Components:** `PascalCase.tsx` in French — `CarteUtilisateur.tsx`,
  `FormulaireUtilisateur.tsx`.
- **Hooks:** `use` + French PascalCase — `useFiltreUtilisateurs.ts`.
- **Server Actions:** French verb naming, same as any mutation function —
  `creerUtilisateur.ts`.
- **Mappers / data-access functions:** `obtenirListeUtilisateurs.ts`,
  `creerUtilisateur.ts`.
- **Test files:** `*.test.ts(x)`, subject first, in French — but the
  description string inside stays English (see Language Policy above).

### Class and Type Naming Rules

- `PascalCase`, no `I`/`T` prefixes.
- Domain entities: `Utilisateur`.
- DTOs: `<Concept>Dto` — `UtilisateurDto`.
- Frontend contracts: `<Concept>Reponse` / `<Action><Concept>Entree` —
  `UtilisateurReponse`, `CreerUtilisateurEntree`.

### Method and Function Naming Rules

French verbs, `camelCase`:

- **Queries:** `obtenir`, `lister`, `trouver` — `obtenirListeUtilisateurs`.
- **Mutations:** `creer`, `mettreAJour`, `supprimer` — `creerUtilisateur`.
- **Transformations:** `mapper`, `construire`, `formater`, `filtrer`,
  `valider`.
- **Boolean checks:** `est`, `a`, `peut`, `doit` — `aPermission`.
- **Prop callbacks** keep the React `on` prefix as-is — `onSoumission`.

### Variable Naming Rules

- `camelCase`, French.
- Booleans: `est`, `a`, `peut`, `doit`, `en` — `peutConsulter`,
  `chargementEnCours`.
- Collections: plural French nouns — `utilisateurs`.
- Identifiers keep the `Id` suffix as-is: `utilisateurId`.
- Dates: `creeLe`, `misAJourLe`.

### Enum and Constant Naming Rules

- Constants: `UPPER_SNAKE_CASE`, French words — `TAILLE_PAGE_PAR_DEFAUT`.
- Permission keys are the backend's own codes, used verbatim, grouped
  under an `UPPER_SNAKE_CASE` object — `PERMISSIONS_UTILISATEURS`.
- Contractual enum values keep their actual coded value; the display
  label is French, in a separate module-owned mapping.

### i18n Key Naming Rules

- Translation objects use French camelCase keys **and** French string
  values.
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
data-fetching patterns, testing organization, performance budgets —
applies identically whether a project uses this appendix or not. Only the
vocabulary changes.
