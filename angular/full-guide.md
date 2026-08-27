# Angular Enterprise Architecture Guide — Improved Edition

**Production-grade architecture for large, long-lived enterprise frontend systems (ERP-grade).**

- **Audience:** senior engineers, mid-level engineers, new joiners, and architecture reviewers.
- **Scope:** a single Angular SPA monorepo (Nx) fronting an enterprise backend (e.g., Django REST / OpenAPI), designed to survive growth to 20+ business domains, hundreds of features and screens, many teams, multi-company/branch/warehouse operations, financial workflows, real-time notifications, document management, huge tables, Arabic/French/English with RTL, and years of production life.
- **Rule vocabulary:** `MUST` / `MUST NOT` = enforceable, checked in review or CI. `SHOULD` = default expectation; deviations need justification. `MAY` = optional, only when justified by concrete need.
- **Positioning:** ERP is the flagship illustration, but every rule applies to any large enterprise system — logistics, healthcare admin, banking portals, field-service suites. Nothing here assumes "accounting" specifically.

> **Review note:** the baseline guide referenced in the request (`angular_erp_architecture_guide.md`) was reviewed against the structure it proposes: flat type-based folders (`core/`, `shared/`, `design-system/`, `domain/`, `feature/`, `data-access/`, `ui/`, `util/`) on Nx + Angular Material/CDK. All findings below target that baseline and the generic-companion assumptions typically shipped with it.

---

## Table of Contents

**Part I — Baseline Architecture Review**
- R1. Review Method
- R2. Verdict Summary (Keep / Change / Remove / Add)
- R3. Detailed Findings
- R4. Systems-Perspective Evaluation

**Part II — The Improved Architecture**
1. Architectural Principles
2. System View & Growth Model
3. Workspace Structure & Public APIs
4. Configuration Architecture
5. Cross-Cutting Concerns Map
6. Authentication, Authorization & Security
7. HTTP & API Architecture
8. State Management Rules
9. Nx Enforcement: Making Boundaries Real
10. Design System
11. ERP-Specific Architecture Concerns
12. Performance Defaults
13. Testing Architecture
14. Observability
15. Maintainability & Abstraction Policy
16. Library Stack Review

**Part III — Reference Artifacts**
- C. Final Folder Structure
- D. Dependency Rules
- E. Centralization Rules
- F. Library Stack Decision Table
- G. Feature Implementation Checklist
- H. Architecture Red Flags

**Final Recommended Architecture**

---

# Part I — Baseline Architecture Review

## R1. Review Method

Each finding lists four things:

1. **What is wrong or incomplete** in the baseline.
2. **Why it matters in a large ERP** (20+ domains, hundreds of devs over time).
3. **What to do instead**.
4. **Severity**: whether adopting it is **MUST** (mandatory) or **SHOULD** (strongly recommended, deviations allowed).

No finding below was invented for volume. Every one corresponds to a failure mode that actually occurs when these systems reach 100k–500k LOC of frontend code, or to a hard requirement stated in the project brief (security, i18n/RTL, observability, contract-first API, ERP operational contexts).

## R2. Verdict Summary

### Keep

| Baseline decision | Why it stays |
|---|---|
| Type separation between `feature` / `data-access` / `ui` | Correct instinct: separates orchestration from data fetching from presentation. Kept, but as the *second* axis inside each domain, not as global top-level buckets. |
| A distinct design-system library on top of Material + CDK | Correct strategy: Material/CDK are the right primitives layer; wrapping only branded surfaces keeps upgrade cost low. |
| Library-based Nx workspace instead of one giant app folder | Correct foundation for boundaries, caching, affected builds. |
| Lazy-loaded routed features | Correct default at this scale. Tightened (preload policy, budget gates) in §12. |
| Signals-oriented UI direction | Correct for Angular 17+ era; formalized into state rules in §8. |
| Facades "where justified" skepticism | Good anti-ceremony instinct; made precise in §15. |

### Change

| # | Baseline behavior | Required change |
|---|---|---|
| 1 | Flat top-level type folders (`feature/`, `data-access/`, `ui/` …) | Reorganized **domain-first**: `libs/domains/<d>/{api,domain,data-access,ui,feature-*}` — see §3. |
| 2 | `core/` as singleton-services home | Split into **focused platform libraries** (`platform/config`, `platform/http`, `platform/auth`, `platform/contexts`, …); "core" as a concept dissolves. |
| 3 | `shared/` + `util/` catch-all utilities | Capped, governed `shared/` with explicit admission criteria and per-topic util libs; `util/` bucket removed. |
| 4 | Reading `environment.ts` directly in feature code | Full runtime-config pipeline: fetch → schema validation → validated `AppConfig` → injection token only (§4). |
| 5 | Cross-domain imports allowed anywhere | Only via each domain's single published `api` library; all internals become lint-inaccessible (§9). |
| 6 | Guards treated as authorization | Guards demoted to UX affordance; backend declared the sole security boundary (§6). |
| 7 | Ad-hoc error handling per component | Centralized `ApiError` normalization + user-facing message policy matrix (§7.6). |
| 8 | Untyped ad-hoc HTTP clients | Contract-first OpenAPI-generated types consumed by thin domain clients (§7.2). |
| 9 | Free-form store usage | State decision ladder: local → URL → SignalStore (feature) → platform stores; no god stores (§8). |
| 10 | Environment files as deployment configuration | Build once, deploy anywhere: runtime `runtime-config.json` rendered per environment, artifact stays immutable (§4.5). |

### Remove

| Removed thing | Reason |
|---|---|
| Top-level `util/` bucket | It reliably becomes a random dump with unknown ownership; replaced by scoped, named utility libraries with admission rules (§15.3). |
| Global `domain/` models folder holding "all entities" | Entities belong to their domain library; a global model folder forces cross-domain coupling through the back door. |
| "Shared components" folder without admission criteria | In practice every team parks half-finished widgets there. Access now requires the checklist in §15.3. |
| Per-environment build-time `environment.prod.ts` variant matrix as the config mechanism | Unscalable and leak-prone; superseded by the runtime pipeline (§4). |
| Any notion that hidden UI or disabled buttons constitute access control | Security-relevant removal; see §6.8. |

### Add

| Added pillar | Section |
|---|---|
| Configuration pipeline (loader → schema validation → `AppConfig`) | §4 |
| Platform context services (company / branch / warehouse / fiscal period / currency) with HTTP propagation & cache versioning | §5.4, §11.1 |
| Correlation-ID spine linking browser ↔ API ↔ background jobs | §14.3 |
| Centralized logging / telemetry / Web Vitals reporting | §14 |
| Permission service + directive fed by backend grant list (never frontend-invented roles) | §6.5 |
| Contract-first API typing with CI drift check | §7.2 |
| Long-running operation client pattern (job polling / SSE bridged to notifications) | §11.6 |
| Standardized file upload/download surfaces | §5.3 |
| Runtime locale switching + RTL strategy (ar/fr/en) | §5.5 |
| Large-table defaults (server-side paging thresholds, virtualization) | §12.3 |
| Testing pyramid with golden-path E2E and integration tests over generated contracts | §13 |
| Tagging + ESLint boundary enforcement so forbidden imports are impossible, not discouraged | §9 |
| Implementation checklist + red-flag catalog | Artifacts G, H |

## R3. Detailed Findings

### Boundaries & Structure

**RF-01 · Type-first top-level folders do not scale to multiple domains.**
*Why it matters:* with 20+ domains, `feature/` contains 200 sibling libraries whose ownership is invisible; any developer can import anything from any other domain silently. Coupling grows invisibly until a one-line change in accounting breaks sales reports.
*Instead:* domain-first tree; type becomes a second-level segment inside each domain; cross-domain access only through `domains/<d>/api`. **MUST**

**RF-02 · No enforcement mechanism existed at all.**
*Narrative rule* ("keep domains independent") decays under deadline pressure within weeks.
*Instead:* Nx module boundaries + tags + per-directory ESLint bans make violations fail locally and in CI (§9). Prefer prevention over documentation. **MUST**

**RF-03 · Public-API discipline undefined.**
Without barrel-file discipline, deep-imports appear naturally, making refactors impossible ("change ripple hits nobody knows which files").
*Instead:* every library exports exactly one entry point; deep relative imports banned by lint; internal directories declared `private` by convention and by lint rule (§3.4, §9). **MUST**

### Configuration

**RF-04 · Direct `environment.ts` reads across features.**
Kills testability (hard to vary settings), makes prod-vs-stage differences compile-time (violates build-once-deploy-anywhere), scatters knowledge of URLs/flags/tuning knobs.
*Instead:* single approved path — loader → zod schema validation → `APP_CONFIG` token → grouped typed accessors (§4). Lint bans `environments/*` imports outside `platform/config`. **MUST**

**RF-05 · No stance on what a frontend secret is.**
The baseline implies env files may hold sensitive values.
*Instead:* explicit rule — nothing shipped to the browser is secret; enumerate never-shippable values; separate public-runtime-values template maintained by platform owners (§4.7). **MUST**

### Security

**RF-06 · No session/token lifecycle design.**
ERPs hold long sessions with sensitive data; refresh races and silent expiry cause both support tickets and lockouts mid-document-entry.
*Instead:* OIDC Authorization Code + PKCE, single-flight refresh queue, pre-expiry renewal, logout cleanup defined centrally (§6.2–6.4). **MUST**

**RF-07 · Guard-as-security ambiguity.**
If the baseline lets anyone believe guards "protect" routes, someone will ship a page relying only on hiding menu entries.
*Instead:* documented principle — *frontend authorization is UX, not a security boundary*; every route guard decision mirrored server-side; permission checks drive *affordances*, never trust (§6.5–6.8). **MUST**

**RF-08 · XSS/redirect/dependency surface unaddressed.**
Large systems accumulate: innerHTML shortcuts, redirect params accepted blindly, stale vulnerable transitive deps.
*Instead:* sanitization policy, redirect allowlist, external-link hygiene, SCA automation (Renovate + audit gate + license allowlist) (§6.9–6.11). **MUST** (SCA: MUST in CI)

### HTTP / API

**RF-09 · No pagination/filter/sort contract.**
Every domain invents its own param names & shapes; tables can't share infrastructure; dashboards can't reuse query builders.
*Instead:* standardized `Page<T>` envelope + query-param conventions + reusable request-builder types (§7.4). **MUST**

**RF-10 · Race conditions unhandled in search/list patterns.**
Classic ERP bug: fast typist sees stale results overwrite newer ones.
*Instead:* standard search-store pattern (debounce + cancellation via `rxResource`/`switchMap`, key-based revalidation) packaged once in shared infra (§7.7). **MUST** for searchable lists

**RF-11 · Retry/caching/dedup semantics unspecified.**
Result: either duplicate storms (N identical requests on dashboard) or unsafe retries mutating orders twice.
*Instead:* interceptor-level retry restricted to idempotent verbs/timeouts; dedup via `shareReplay` only for reference-data loaders; mutations never retried transparently (§7.5). **MUST**

### State

**RF-12 · No state-placement doctrine.**
Baseline silence produces both extremes: god stores and scattered duplicated state.
*Instead:* six state categories with placement rules, URL-as-source-of-truth for list state, effects discipline, SignalStore as the sanctioned shared-state tool; no NgRx Store/Effects unless justified per-team later (§8). **MUST**

### i18n / RTL

**RF-13 · Multi-language incl. Arabic + RTL was listed but not designed.**
Locale switching needs to be *runtime* (same immutable artifact), lazy per-domain translation scopes keep bundles sane, and RTL must be automatic from a single `dir` root rather than per-component hacks.
*Instead:* Transloco scope-per-domain, ICU plurals, central LocaleService owning language→dir→formatting chain, Intl wrappers as sole date/number/money formatters (§5.5). **MUST** given ar/fr/en requirement

### Observability

**RF-14 · No correlation ID strategy.**
Supporting "the report printed twice for Company A but not B at 14:05" without request linkage is archaeology.
*Instead:* UUID correlation id minted at first request per user action (or continued via header), attached everywhere including WebSocket events; surfaced in errors shown to users for reference (§14.3). **MUST**

**RF-15 · Logging = console.log scatter.**
Uncontrolled logs leak payloads and produce noise; nothing correlated survives to a sink.
*Instead:* leveled structured logger with redaction filters and remote sink gated by config; global window handlers; Web Vitals collection (§14.1–14.2). **MUST** (remote sink: SHOULD initially)

### Testing

**RF-16 · Coverage-% culture instead of risk-based pyramid.**
Blind coverage yields brittle template tests while missing financial-rule bugs.
*Instead:* pyramid defined by responsibility table; mandatory tests enumerated (business rules, permission matrices, store reducers/facades behaviors, error normalization, golden-path E2E); implementation-detail testing explicitly rejected (§13). **MUST**

### Performance

**RF-17 · "Lazy loading" only.**
At ERP scale, the killers are big grids rendering 5,000 rows, dashboards firing 40 parallel requests, bundle drift.
*Instead:* architectural defaults: server-side paging mandate above threshold rows, CDK virtual scrolling for wide lists, budgeted initial bundles with CI assertion, selective preloading, dedup on shared endpoints, `OnPush`/zoneless-ready signals defaults (§12). **MUST** (defaults)

### Operational ERP concerns

**RF-18 · Company/branch/warehouse/fiscal/currency context missing entirely.**
These contexts affect nearly every screen, query payload, cache identity and action legality. Without centralized handling, each feature hand-rolls `companyId` threading, caches go stale on switch, and postings hit the wrong fiscal year.
*Instead:* platform `contexts` group: reactive signal stores per context + HTTP propagation interceptor + context-version token woven into data-store keys + guarded switch workflow with dirty-state confirmation (§5.4, §11.1). **MUST**

**RF-19 · Files/documents uploaded ad hoc.**
Attachment upload/download appears in 30 screens; every implementation handles progress/errors/sanitization differently.
*Instead:* single `FilesService` + reusable attachments panel component; presigned-flow friendly; size/type gates configurable through AppConfig (§5.3). **MUST** before second consumer

**RF-20 · Long-running operations have no client pattern.**
Postings, report generation, batch imports exceed HTTP timeouts; naive calls hang or fail confusingly.
*Instead:* job submission → `{operationId}` → status polling w/ exponential backoff (or SSE) integrated with notification center and router-aware resume (§11.6). **MUST** before first long op

**RF-21 · Document status machines rendered ad hoc.**
Order/invoice/voucher statuses drive actions, colors, allowed transitions. Divergent copies rot quickly.
*Instead:* status metadata sourced from backend/domain model drives timeline/badges/action availability via one shared renderer + typed transition helpers (§11.3). **SHOULD**

**RF-22 · Optimistic updates unconstrained.**
Fast-looking but dangerous on financial documents; conflicts mishandled.
*Instead:* optimistic writes only for whitelisted low-risk entity fields with rollback-on-error and ETag/version conflict surfacing (§7.8, §11.5). **SHOULD**

## R4. Systems-Perspective Evaluation

Answering the brief's question — does the *baseline* remain maintainable when the ERP grows? — by criterion:

| Criterion | Baseline verdict | Decisive factor |
|---|---|---|
| Coupling | ❌ Fails | Type-buckets enable invisible cross-domain imports; no machine-checked boundary. |
| Cohesion | ⚠️ Partial | Within-feature layering good; globally, cohesion weaker than domain grouping. |
| Dependency direction | ⚠️ Unmanaged | No declared graph; direction is convention only. |
| Change impact | ❌ Poor | Refactors fan out through deep imports; impact analysis manual. |
| Scalability (people) | ❌ Poor | Ownership unclear; merge hotspots in `core/`, `shared/`; onboarding slow. |
| Testability | ⚠️ Partial | Layered shape helps; env-direct reads and scattered singletons hurt. |
| Build performance | ✅ OK→excellent with Nx affected | Must add explicit affected-CI contract (§9.6). |
| Runtime performance | ⚠️ Unknown | Defaults needed (§12) else grid/dashboard regressions ship. |
| Developer experience | ⚠️ Mixed | Easy start; heavy cost after ~10 domains. |
| Upgradeability | ⚠️ Partial | Nx helps; wrap-count on Material kept deliberately low helps (§10). |
| Operational diagnosability | ❌ Fails today | No telemetry/correlation story (fixed by §14). |

**Overall:** the baseline is a reasonable *starting skeleton* for a small app and materially better than chaos, but it is not yet an *architecture* — several load-bearing walls (boundaries, config, security posture, observability, operational contexts) are either decorative or missing. Part II rebuilds them.

---

# Part II — The Improved Architecture

## 1. Architectural Principles

These are the load-bearing rules. Everything else in this document is their mechanical consequence. When a design dispute arises and no specific rule covers it, resolve it against these principles in order.

**P1 — Boundaries are enforced, not suggested.**
A rule without a machine check is a suggestion, and suggestions do not survive deadline pressure at scale. Every MUST below is paired with an enforcement mechanism: Nx module boundaries, ESLint restrictions, CI gates, or checklist steps. Rules that cannot yet be automated stay visible in the feature checklist (Artifact G) and PR template.
> *Enforcement principle: developers should be prevented from introducing forbidden dependencies rather than merely being told not to.*

**P2 — Domain-first ownership; type is secondary.**
The primary axis of the codebase is *who owns it* (sales, inventory, purchasing, accounting…). The type (`feature`, `data-access`, `ui`, `domain`) is the second axis inside each domain. Cross-domain access exists only through each domain's published `api` library. This makes "which team must I talk to?" answerable from an import path alone.

**P3 — The platform owns cross-cutting reality; domains own business behavior.**
Configuration, authentication session, HTTP infrastructure, error normalization, logging/telemetry, notifications, files, i18n formatting, and operational contexts are implemented **once** under `platform/`. Feature teams consume them via public APIs and never reimplement them. Conversely, the platform contains zero business logic — sales rules live only in sales.

**P4 — Configuration flows one way, through one door.**
Environment/runtime values → config loader → schema validation → validated immutable `AppConfig` → typed injection tokens. Feature code never reads environment files, `import.meta.env`, or raw query strings for settings. (§4)

**P5 — Frontend authorization is UX, not security.**
Guards, hidden buttons and disabled menus shape experience and reduce noise; they establish nothing about what is actually permitted. Every permission decision is re-verified server-side. Nothing shipped to the browser is secret. (§6)

**P6 — Server state belongs to the server; the client caches it, it does not own it.**
Lists, documents and reference data are cached client-side with explicit invalidation hooks; the URL owns navigational state (filters/sort/page); components own purely visual state. There is exactly one source of truth per piece of state. (§8)

**P7 — Simplicity over ceremony.**
Introduce an abstraction because it protects a boundary or removes meaningful duplication — not because abstraction looks architectural. When two designs tie, choose the one with fewer concepts, fewer files, fewer indirection hops. Micro-frontends, event-bus architectures, and plugin systems require a concrete organizational driver, and none exist here by default. (§15)

**P8 — Consistency beats local cleverness across hundreds of screens.**
Grids, forms, dialogs, toasts, error presentation, loading states, money/date rendering and empty states behave identically everywhere because they come from shared surfaces. A feature inventing its own dialog/upload/paging convention is a defect even if its code is pretty. (§10, §11)

**P9 — Observability is part of the architecture, not an add-on.**
Every user action can be traced browser→API→worker through correlation IDs; errors arrive normalized and grouped; key Web Vitals and workflow timings flow to a sink. If you cannot debug it in production, it is not done. (§14)

**P10 — Performance defaults are architectural; tuning is opportunistic.**
Safe-by-default thresholds (virtualization row counts, budgets, page sizes) ship with the skeleton so slow patterns cannot become endemic. Premature micro-optimization is prohibited until measured. (§12)

### Minimal vocabulary used throughout

| Term | Meaning |
|---|---|
| **Platform library** | Domain-free infrastructure library under `libs/platform/*` |
| **Domain** | One business area (sales, inventory, purchasing, accounting, administration…) owning its features end-to-end |
| **Domain API lib** | The single outward-facing library of a domain; everything else inside a domain is private |
| **AppConfig** | The validated, immutable runtime configuration object injected at bootstrap (§4) |
| **Context** | Operational scope actively selected by the user/workspace: company, branch, warehouse, fiscal period, currency (§5.4) |

## 2. System View & Growth Model

### 2.1 Deployment & runtime shape

One Angular SPA ("shell") served from CDN/static hosting, talking HTTPS JSON+multipart to a versioned REST API (OpenAPI-published), plus SSE/WebSocket for notifications. No BFF by default; a thin reverse proxy may normalize URLs/headers.

```text
┌─────────────────────────────────────────────────────┐
│ Browser SPA (immutable artifact + runtime-config.json)
│   shell ─ lazy chunks: sales/* inventory/* accounting/*
│   platform: config auth http contexts i18n observability
│   design-system (Material/CDK wrapped)                │
└───────────▲──────────────────────▲───────────────────┘
            │ HTTPS JSON/multipart │ SSE / WSS
            ▼                      ▼
┌───────────────────────────┐  ┌──────────────────┐
│ API gateway / Django REST │  │ Notification hub │
│ /api/v1 … OpenAPI schema  │◄─┘ (bridges Celery    │
│ enforces authorization    │      job events)     │
└─────────┬─────────────────┘  └──────────────────┘
          ▼
   PostgreSQL / workers / object storage
```

Deliberate non-decisions:
- **No micro-frontends.** Route-level lazy chunks inside one Nx app give equivalent isolation for organizational purposes until there is a concrete need (independent deployment cadence between org units that admin boundaries actually split). Revisit only when documented pain appears. **MUST NOT** introduce MFEs otherwise.
- **No frontend event bus.** RxJS streams within services + signals are sufficient; a global pub/sub creates hidden coupling worse than imports. **MUST NOT** without architectural approval.
- **No SSR requirement initially.** Behind-auth ERP screens gain little; revisit dashboards/public landing pages independently. MAY later.

### 2.2 How the architecture absorbs growth

Each stress vector from the brief maps to a concrete mechanism:

| Growth vector | Mechanism that absorbs it | Where |
|---|---|---|
| 20+ domains | Domain tree + per-domain api lib; adding a domain = scripted scaffold + 2 file edits (routes, nav) | §3, §9 |
| Hundreds of devs/teams | Ownership encoded in tags/CODEOWNERS per domain folder | §9.5 |
| Very large tables | Server-side paging mandate, virtualization thresholds, saved views | §12.3 |
| Complex permissions | Central permission service fed by grants; route canMatch + directive layers | §6.5 |
| Multi-company/branch/warehouse/fiscal/currency | Context stores + propagation interceptor + cache versioning | §5.4, §11.1 |
| Financial workflows | Document state-machine renderer, confirm-and-reason pattern for posting actions | §11.2–11.4 |
| Real-time notifications | Single stream service → toast + notification center + targeted cache invalidation | §5.2, §11.7 |
| Background jobs exposed via APIs | Operation client pattern (poll/SSE resume) | §11.6 |
| Files/documents | FilesService + attachments panel; presign support | §5.3 |
| ar/fr/en + RTL | Transloco scopes + LocaleService chain (lang → dir → Intl formatters) | §5.5 |
| Frequent backend changes | Contract-first generated types + CI drift gate + deprecation policy | §7.2 |
| Long-lived deployments | Build-once/runtime-config; SCA automation; upgrade cadence | §4.5, §16.4 |

### 2.3 Evaluation matrix for future decisions

Any contested choice gets scored on: coupling · cohesion · complexity · scalability · performance · security · testability · developer experience · upgradeability · operational cost. This document already applied that scoring to produce these calls; copy the method rather than argue taste.

## 3. Workspace Structure & Public APIs

### 3.1 Final shape of `libs/`

```text
apps/
  erp-shell/            # the single SPA: bootstrap, routing root, global providers, layout chrome
  e2e/                  # Playwright project(s)

libs/
  platform/                       # domain-free infrastructure — each child a separate Nx lib
    config/                       # loader + schema + AppConfig tokens (§4)
    http/                         # interceptors, ApiError model, retry policy (§7)
    auth/                         # session lifecycle, OIDC glue, token state (§6)
    permissions/                  # grant loading, has() semantics, directive, guard factory
    contexts/                     # company/branch/warehouse/fiscal/currency stores (§5.4)
    i18n/                         # Transloco root config, LocaleService, Intl wrappers (§5.5)
    notifications/                # toast service, stream bridge, notification center API
    dialogs/                      # confirm/prompt/workflow-dialog standards on Material Dialog
    files/                        # upload/download/presign client, attachments panel API (§5.3)
    operations/                   # long-running job client pattern (§11.6)
    logging/                      # leveled logger, redaction filters (§14.1)
    telemetry/                    # correlation ids, Web Vitals, sink adapters (§14.2–14.4)
    feature-flags/                # flag source = AppConfig + remote overrides (§4.8)

  design-system/                  # tokens, theming, wrapped components, RTL/a11y guarantees (§10)

  shared/                         # small, governed, domain-free surface — capped intentionally
    ui/                           # generic presentational widgets that survived admission review
    utils/                        # pure TS helpers by topic: money, dates, fp-lite …
    testing/                      # shared test harnesses/fixtures/fakes for platform APIs

  domains/
    sales/
      api/                        # ⟵ THE public face of sales (models+facade+events), tagged public
      domain/                     # entities, value objects, business rules, status machines
      data-access/                # generated-types consumers, endpoints, mappers, caches
      ui/                         # widgets reused across multiple sales features
      feature-orders/             # routed feature libs (one per meaningful route group)
      feature-customer-360/
    inventory/
      api/
      domain/
      data-access/
      ui/
      feature-stock-ledger/
      feature-transfers/
      feature-cycle-counts/
    purchasing/
      …
    accounting/
      api/
      domain/
      data-access/
      ui/
      feature-ledger/
      feature-fiscal-periods/
      feature-payment-runs/
    administration/
      api/
      domain/
      data-access/
      ui/
      feature-users-and-roles/
      feature-context-setup/
```

Rules:
- **MUST** one new library per routed feature area (`feature-orders`), not per screen-file; a "feature lib" can contain several sibling routes if they share models & navigation context.
- **MUST NOT** create `shared/ui` or `shared/utils` entries without passing the admission criteria in §15.3.
- **MUST NOT** add a business concept to `platform/*`. If it names a noun from the business (invoice, order, tax code), it belongs to a domain.
- **SHOULD** keep `domains/<d>/ui` empty until ≥2 features inside the domain genuinely need the same widget.

### 3.2 What lives where — responsibility table

| Library | Owns | Never contains |
|---|---|---|
| `app shell` | bootstrap order (config → auth → contexts → i18n → telemetry), root routes, app chrome | routing internals of domains; services |
| `platform/config` | runtime-config fetch+validate+tokens | any env-specific default invented ad hoc |
| `platform/http` | interceptors, ApiError mapping, retry rules, request metadata contract | endpoint functions |
| `platform/auth` | token/session lifecycle, user identity signal, logout flow | permission *semantics* (that's permissions/) |
| `platform/permissions` | grants shape, `has()` evaluation, directive/guards | which screens exist |
| `domain/<d>` | entities, status machines, business-rule pure functions, DTO↔UI types | HTTP calls |
| `data-access` (per domain) | endpoint functions over shared HttpFacade, mappers, list-state stores, invalidation wiring | UI decisions, translation strings |
| `feature-*` | routed pages composition, forms orchestration, local signals, translation scopes | other domains' imports |
| `<d>/api` | published facade: read models, command results, events, small selector set | deep internals of its own domain |

### 3.3 Realistic feature anatomy

Concrete example — **Sales Order management** (`feature-orders`):

```text
libs/domains/sales/
  api/src/index.ts                     # exports SalesOrdersApi, OrderView, OrderEvent… ONLY
  domain/
    src/lib/models/order.ts            # Order entity & statuses (Draft→Confirmed→Picked→Shipped→Invoiced)
    src/lib/rules/order-rules.ts       # PURE functions: canConfirm(order), validateLines(), credit-check gate
    src/lib/orders-status-machine.ts   # transition table consumed by shared StatusFlow renderer (§11.3)
    index.ts                           # private within sales (importable only by sales/*)
  data-access/
    src/lib/orders.api.ts              # getOrders(page:OrderQuery): Observable<Page<OrderDto>>
    src/lib/order.mapper.ts            # DTO ⇄ view-model (money minor-units, dates, status enums)
    src/lib/orders.store.ts            # SignalStore: list/page/filters keyed w/ context version (§8.4)
    src/index.ts                       # private to sales
  feature-orders/
    src/lib/pages/orders-list/         # grid page: URL-synced filter chips, virtualized rows
    src/lib/pages/order-editor/        # typed reactive form; dirty-guarded
    src/lib/pages/order-detail/        # header card + lines + timeline + actions bar
    src/lib/orders.routes.ts           # exported through api? NO → exported only into shell via api-less route hook (see §3.4)
    src/index.ts
```

Route exposure exception: the **shell needs the lazy route manifest**. Each feature lib exports `routes: Routes`; each domain's `api` lib re-exports a single `SALES_ROUTES` composed tree. Nothing else about feature internals crosses out.

### 3.4 Public API discipline

- **One entry point:** every lib exposes exactly one `index.ts` (Nx secondary-entry point for domains' internal sharing: e.g., `@erp/sales/domain`). Deep paths like `@erp/sales/data-access/orders/store` are lint-forbidden everywhere outside their own lib.
- **Minimal surfaces:** a barrel re-exporting everything is a boundary failure wearing a costume. Each `api` lib reviews exports like a public SDK: remove-before-add.
- **Private conventions declared:** internal folders use `_internal/` prefix; Nx sets tags so no other project may depend on the lib except its domain siblings.
- Version-free visibility rule: instead of semver ceremony inside one repo, ownership gates changes — PR touching another team's api lib requires CODEOWNERS approval of that domain.

## 4. Configuration Architecture

> **Enforced rule: Feature code must never read environment variables directly.**
> All runtime configuration is consumed exclusively through validated, typed injection tokens. Direct imports of `environments/*` outside `platform/config` fail the build (lint). Direct reads of `location.search`/`window.ENV`/`import.meta.env.*` for settings are review-blocking defects.

### 4.1 Why this rule exists

Compile-time environment files (`environment.prod.ts`) tie every setting to every build: you cannot change an API base URL after build without recompiling, QA cannot point at staging data from the same artifact, and one careless commit ships a production flag everywhere. They also invite secrets into the repo and bundle. A single validated pipeline fixes all of it: **build once, deploy anywhere**, with every value passing through schema validation before any service sees it.

### 4.2 Build time vs runtime

| Category | Examples | Mechanism |
|---|---|---|
| Legitimately static | app version/commit hash, Sentry-ish public DSN *if* deliberately public, base HREF, client id (public, PKCE) | injected at build via `define`-style replacement or generated `version.ts`; still only readable through `AppConfig` |
| Everything else | API base URL per deployment, default locale, page-size defaults, upload caps, currency/locale options, flag defaults, observability sampling, portal URLs | runtime JSON fetched pre-bootstrap |

**MUST NOT** differentiate builds by environment beyond immutable metadata + asset swaps.

### 4.3 The approved pipeline

```text
assets/runtime-config.json   (+ optional ?cfg= override for test rigs)
        ↓ fetch() in main.ts BEFORE bootstrap
      raw object
        ↓ platform/config loader: deep-merge profile defaults → env payload
      merged draft
        ↓ zod schema validation  → produce frozen readonly AppConfig
   ok: provideConfig(appConfig) → bootstrapApplication()
   bad: boot-fail screen listing missing keys + correlation hint (prod), loud error (dev)
        ↓
APP_CONFIG InjectionToken<AppConfig>          (single root provider, Object.freeze'd)
        ↓
typed grouped accessors  inject(ApiConfig) / inject(UploadsConfig) / inject(ObsConfig) …
        ↓
application code (features/domains/design-system/shared)
```

### 4.4 Reference implementation (`libs/platform/config`)

```ts
// app-config.schema.ts
import { z } from 'zod';

export const RuntimeSchema = z.object({
  api: z.object({
    baseUrl: z.string().url(),
    requestTimeoutMs: z.number().int().positive().default(30_000),
    versionPrefix: z.string().regex(/^v\d+$/),
  }),
  uploads: z.object({
    maxFileMb: z.number().positive().default(25),
    acceptMime: z.array(z.string()).default(['image/*','application/pdf']),
  }),
  locales: z.object({
    available: z.array(z.enum(['ar','fr','en'])).min(1),
    fallback: z.enum(['ar','fr','en']).default('en'),
    defaultCurrency: z.string().length(3),
  }),
  flags: z.record(z.boolean()).default({}),
  obs: z.object({
    logLevel: z.enum(['debug','info','warn','error']).default('info'),
    sampleRate: z.number().min(0).max(1).default(0.2),
    remoteSinkUrl: z.string().url().optional(),
  }),
});
export type RuntimeConfig = z.infer<typeof RuntimeSchema>;
```

```ts
// load-runtime-config.ts  — runs inside main.ts before bootstrapApplication
export async function loadRuntimeConfig(): Promise<RuntimeConfig> {
  const url = new URL(location.href);
  const cfgPath = url.searchParams.get('cfg') ?? 'assets/runtime-config.json';
  const res = await fetch(cfgPath, { cache: 'no-store' });
  if (!res.ok) return bootFail(`runtime-config.json ${res.status}`);
  const parsed = RuntimeSchema.safeParse(await res.json());
  if (!parsed.success) {
    return bootFail(parsed.error.issues
      .map(i => `${i.path.join('.')}: ${i.message}`).join('\n'));
  }
  return Object.freeze(Object.fromEntries(
    Object.entries(parsed.data).map(([k,v]) => [k, Object.freeze(v)])
  )) as RuntimeConfig;
}
```

```ts
// tokens.ts
export const APP_CONFIG = new InjectionToken<Readonly<AppConfig>>('APP_CONFIG');
export const ApiConfig     = injectCfg(c => c.api);       // thin helper over APP_CONFIG
export const UploadsConfig = injectCfg(c => c.uploads);   // each returns config['api'], etc.
export const Flags = /* signals-backed map from config.flags */;
```

Consumption in features:

```ts
private readonly api = inject(ApiConfig);
this.http.get(`${this.api.baseUrl}/orders`)  // ✅ approved path
// ❌ import { environment } from '@erp/environments';            → lint error
// ❌ location.search.split('?')[1].includes('debug')            → review defect
```

### 4.5 Deployment-specific configuration

CI builds exactly one artifact per release. Deployment renders `runtime-config.json` from its own secret/config store (K8s ConfigMap mount, S3 object, nginx sidecar volume…). Changing config never requires a rebuild; the artifact stays content-addressed and verifiable. Template of deployable keys lives beside the schema (`runtime-config.template.json`) so drift between schema and deployment docs fails CI when either changes alone.

### 4.6 Missing / invalid configuration behavior

- **Validation failure = boot failure.** The app must not start half-configured: wrong `baseUrl` discovered on first click costs 10× more support time than a blank boot screen listing the exact offending key.
- Boot screen shows key paths and problem messages; never dumps values (values may reveal internal hosts).
- Unit suite executes the schema against the committed production template — CI catches misconfigurations in PRs, not deployments. **MUST**

### 4.7 What may NEVER be treated as frontend configuration/secrets

Anything reachable by the browser is public, full stop — including "hidden" fetches, base64 blobs, and values behind trivial auth. Therefore:

| Never ship / never read frontend-side | Reason |
|---|---|
| Database credentials, connection strings | server-only |
| OAuth **client secret**, signing/JWT keys, private keys of any kind | PKCE flow needs none |
| Payment-gateway private API keys | PSP integration belongs server-side |
| SMTP credentials, SMS/Twilio keys, internal admin creds | server-only |
| Raw internal network topology unrelated to what SPA calls | reduces recon surface |
| Anything you would refuse to paste publicly | common-sense gate |

Frontend-appropriate values are identifiers/toggles/tuning, all validating against the schema above.

### 4.8 Feature flags

- Source of truth: `config.flags` defaults plus optional backend endpoint for remote overrides (e.g., `/flags?tenant=…&user=…`) loaded once post-auth by `platform/feature-flags`.
- Access: `isOn('new-picking-flow')` signal-backed helper. Flags evaluated at render/action time, **not** baked into bundles.
- Hygiene: every flag has an owner + expiry date recorded next to its definition; stale flags are removal tasks, not furniture. **SHOULD** enforced in quarterly review.

### 4.9 Test & local development configuration

- **Unit/component tests:** factories construct literal configs inline — fast, explicit, no fetch mocking.
- **Local dev:** `proxy.conf.json` for CORS-free same-origin calls; optional gitignored `assets/runtime-config.local.json` for personal prefs (never committed).
- **E2E:** Playwright starts servers with `--cfg=e2e/runtime-e2e.json` via the `?cfg=` hook — the same artifact pipeline as production, not a parallel magic path.

## 5. Cross-Cutting Concerns Map

The governing question for every concern below: **where does it live, who owns it, who may consume it?** If a concern appears in more than one place, one of the places is wrong.

### 5.1 Ownership matrix

| Concern | Library | Owner | Consumers | Centralized ✅ | Domain-specific ❌ |
|---|---|---|---|---|---|
| Auth session & tokens | `platform/auth` | platform team | shell bootstrap, http interceptor, guards | login/renew/logout/token state | none |
| Permissions semantics | `platform/permissions` | platform team | directives, guards, features | grant shape, `has()`, directive | which grants a screen uses |
| HTTP infra & ApiError | `platform/http` | platform team | all data-access libs | interceptors, retry, normalization | endpoint functions |
| API base URL & timeout | `platform/config` | platform team | http layer only | AppConfig accessors | — |
| Correlation IDs | `platform/telemetry` | platform team | logging, errors, users | minting/propagation | tagging with module names |
| Global error handling | `platform/logging` + `platform/http` | platform team | whole app | handler wiring, toast policy mapping | none |
| Logging & telemetry | `platform/logging`, `platform/telemetry` | platform team | everything (via service) | sinks, redaction, sampling | business event names |
| Notifications UI+stream | `platform/notifications` | platform team | all features | toasts, stream connection, center | reacting to domain events |
| Dialogs standards | `platform/dialogs` | platform team | all features | confirm/prompt/workflow dialogs | dialog *content* per flow |
| File upload/download | `platform/files` | platform team | attachment surfaces | upload client, validation, progress | business metadata of documents |
| Long-running ops | `platform/operations` | platform team | posting/report/import flows | job lifecycle pattern | what the job does |
| i18n engine + formatting | `platform/i18n` | platform team | every template, formatter | Transloco root, LocaleService, Intl wrappers | translation content per scope |
| Company/Branch/Warehouse/Fiscal/Currency context | `platform/contexts` | platform team | http interceptor, stores, formatters | context stores, switch workflow, cache versioning | how a screen *uses* context |
| Feature flags source | `platform/feature-flags` | platform team | gates in code | flag loading/isOn() | gating decisions |

### 5.2 Notifications

One SSE/WebSocket connection owned by `platform/notifications`, opened post-auth with correlation ID continuity. Two consumption patterns:
- **Ambient** — toasts + notification center drawer, driven by envelope `{id,type,severity,title,body?,link?,ts}`.
- **Targeted invalidation** — domains subscribe through typed adapters (`on('order.confirmed', e => store.invalidate(e.orderId))`). Domains own their reaction logic; they never touch the socket.

Offline-tolerant: stream drops reconnect with exponential backoff and jitter; missed-event reconciliation via last-seen cursor endpoint. **MUST NOT** open feature-owned sockets.

### 5.3 Files

```text
Upload:    FilesService.upload(bucketCtx, file) → progress 0-100 | result ref
           · MIME/size gates from UploadsConfig        · chunked large files (MAY)
Download:  FilesService.download(ref) → blob → anchor save; filename sanitized server-authoritative
Preview:   presigned GET where backend supports it (never expose storage credentials)
```
Standard surface = `<erp-attachments>` panel component (design-system backed by platform/files): list, drop-zone, progress, virus-scan status placeholder, preview modal. Every document-bearing screen reuses it. **MUST**

### 5.4 Operational context (company · branch · warehouse · fiscal period · currency)

This is the ERP heart of cross-cutting design. Five signal-backed stores under `platform/contexts`:

```text
WorkspaceContext
 ├─ company      { id, code, name, currencyBase }        ← usually required
 ├─ branch       { id, code, companyBillingId }          ← optional per deployment
 ├─ warehouse    { id|null }                             ← inventory-touching roles only
 ├─ fiscalPeriod { id, year, month, status: open|closed|locked }
 └─ currency     displayCurrency override                ← reporting convenience
derived: ctxVersion = hash(company, branch, warehouse, fiscalPeriod.id)
```

Behavioral contract:
1. **HTTP propagation** — interceptor attaches `X-Company-ID`, `X-Branch-ID`, `X-Warehouse-ID`, `X-Fiscal-Year` (+ idempotency keys on writes). Backend remains authoritative; headers are intent statements. **MUST**
2. **Cache identity** — every data-access store/query key includes `ctxVersion`; switching any dimension automatically invalidates transient caches without manual plumbing. **MUST**
3. **Switch workflow** — picking another company: if dirty forms exist → confirm-dialog ("unsaved changes"), then reset transient caches, re-fetch nav counts, return user to landing route; selected scope persisted per-user server-side so reloads restore it.
4. **Action legality** — screens consult contexts: posting buttons disabled when `fiscalPeriod.status !== 'open'` (with explanatory tooltip). Final legality still enforced by backend.
5. **Display only ≠ data scoping** — `currency` changes number rendering but never silently converts stored amounts; conversions are explicit, audit-traceable operations.

### 5.5 Localization, i18n & RTL

Chain of custody for language:

```text
LocaleService.language  (user pref > saved server profile > config fallback)
   ├─ sets <html lang> and dir=rtl|ltr at root            ← single root switch
   ├─ loads Transloco scopes lazily: core + per active domain
   └─ swaps Intl wrapper locale: dates, numbers, money (§11.8)
```

Rules:
- **MUST** runtime switching from one immutable artifact (no per-locale builds).
- **MUST** every user-visible string via `$t('scope.key')`; template literals with English text are defects.
- **MUST** RTL correctness achieved at root only (Material handles component direction); features **MUST NOT** hardcode `left/right` margins — use logical CSS properties (`margin-inline-*`).
- Fonts self-hosted with per-script subsets (e.g., Noto Sans latin + Noto Kufi Arabic via unicode-range) — no CDN dependency. Numbers stay western-arabic or locale-native per config choice, consistent app-wide.
- Translation scope layout mirrors domain boundaries: `locales/<lang>/sales.json` etc., loaded on first navigation into the domain. Missing-key detector runs in CI against union of used keys.

### 5.6 Session management UX details

Session heartbeat/policy lives in `platform/auth`: idle warnings before server-side expiry, auto-renewal windows, "session expired — you have unsaved work" preservation prompt for dirty editors (keep form state in sessionStorage keyed by route hash until re-auth).

## 6. Authentication, Authorization & Security

> **The principle, stated once and never negotiated:**
> *Frontend authorization is not a security boundary. The backend must enforce authorization on every request. Guards, hidden menus, and disabled buttons are UX affordances that shape experience and reduce noise — they confer zero trust.*

### 6.1 Protocol choice

OIDC **Authorization Code flow with PKCE**, public client, no client secret exists anywhere in the SPA (see §4.7). Library: `angular-auth-oidc-client` (Angular-native, silent-renew patterns maintained) or direct `oidc-client-ts` wiring if the wrapper fights the architecture. Auth server roles are fetched with the token response.

Why this over password-in-form: long-lived enterprise SSO needs IdP integration (AD/Azure AD/Keycloak), MFA, session policies — building that by hand is a liability.

### 6.2 Token storage trade-offs (explicit)

| Option | XSS exposure | Notes | Verdict |
|---|---|---|---|
| In-memory only + rotating refresh in `httpOnly; Secure; SameSite=Strict` cookie set by auth server behind same-origin proxy | lowest | needs infra cooperation ("BFF-lite") | **SHOULD** when platform supports |
| In-memory access token + refresh token in memory, renewed via hidden iframe / refresh grant | low (nothing durable to steal) | iframe third-party-cookie caveats → prefer refresh grant w/ rotation | **default** |
| `localStorage` persistence of refresh tokens | high — any injected script exfiltrates silently | acceptable *only* where web workers/iframes can't be used and risk accepted in writing | **MUST NOT** default |

Access token lives in a module-scoped variable inside `platform/auth`; no feature ever touches it. The HTTP interceptor injects it. XSS defense therefore = CSP + sanitization discipline (§6.9) because token theft ≈ account takeover regardless of storage slot.

### 6.3 Renewal & races

Single-flight renewal: concurrent 401s queue behind one refresh promise (`withRefreshMutex`); exactly one retry per original request. Proactive renewal at ~80% of token lifetime avoids mid-submit expiry. If renewal fails → session-expired UX preserving dirty editor state (§5.6), redirect back with return URL validated against allowlist (§6.10).

### 6.4 Logout & cleanup

Full sweep order: revoke token (best-effort), terminate SSE, clear caches/stores/signal maps, wipe sessionStorage leftovers, clear service-worker caches for authenticated APIs, navigate to post-logout route. Timeout-based logout reuses the same routine.

### 6.5 Permissions model

```text
grants: string[]   ← returned by backend after login; NEVER synthesized frontend-side
code format:       <domain>.<resource>[.<sub-resource>].<action>
                   sales.order.approve · accounting.payment.post · inventory.transfer.create
```

API surface (platform/permissions): `has(code)`, `hasAny([])`, `hasAll([])` as signals; `*erpPerm="'sales.order.approve'"` structural directive; `canMatchPermissions('sales.order.*')` guard factory that loads grants before routing starts.

Policies:
- Grants cached with version stamp; refreshed on window focus after long idle, and always invalidated globally after any unexpected 403 (signals stale permission sets). **MUST**
- Hide vs disable: actions the user can never perform → hide; contextual blocks (fiscal period closed, workflow state forbids) → disable + reason tooltip. **MUST** consistency check G.
- The *absence* of UI must match the *absence* of grants so users don't hit walls — but remember §6.8: matching UI does not make it safe, the API makes it safe.

### 6.6 Route protection chain

```text
canMatch[authReady]  →  canMatch[permissions]  →  canActivate[contextRequired]
                              ↓                       (company/fiscal resolved)
                         lazy chunk loads → page renders
```
No route renders before config+auth+permission prerequisites exist. Deep links into financial screens behave identically to navigation flows.

### 6.7 File access

Downloads/uploads pass through `platform/files`, which uses short-lived presigned URLs issued by the API — the SPA never holds bucket credentials, and generated URLs carry user identity for server-side audit.

### 6.8 What guards do NOT protect

Explicit list for reviewers: hidden routes (URLs leak through bundles/history), menu items removed for role X (X may still call endpoint directly), disabled buttons (trivially bypassed), rows filtered from tables (API-level row filtering is what matters), "admin-only" components gated by flags. Every one of these is an affordance; each paired API behavior must independently verify scope + permissions + context headers. Backend checklist item exists in Artifact G for this reason.

### 6.9 XSS posture

- Angular's sanitizer trusted; **MUST NOT** call `bypassSecurityTrust*` on anything data-derived without security review sign-off recorded in PR.
- Rich text rendering (report descriptions etc.) uses allowlist sanitizer; images only from approved origins.
- CSP shipped by host: strict script-src (no `unsafe-eval`; nonces if needed), `object-src 'none'`, frame-ancestors deny, report-uri wired to telemetry sink. Trusted Types MAY be adopted platform-wide once Material pass-through verified.
- No `target=_blank` without `rel="noopener noreferrer"` (lint rule).

### 6.10 Redirects & external URLs

Return URLs validated against origin allowlist (config-driven list of deep-link hosts/paths); absolute external redirects require explicit product approval registry. Wildcard `redirect_uri=*` banned.

### 6.11 Dependency & supply-chain security

Renovate bot weekly updates + lockfile commits (pnpm/npm deterministic installs); CI gate fails on known-vulnerable versions flagged critical/high; license allowlist (MIT/BSD/Apache-2.0/ISC) enforced on all new deps; egress-scoped prod builds (no telemetry SDKs beyond approved sinks).

## 7. HTTP & API Architecture

### 7.1 Layering (dependency flow is downward only)

```text
Component  (presentation; local signals; template)
    ↓ calls
Feature orchestration (forms, dialog sequencing, navigation effects)
    ↓ via
Store / facade — WHERE JUSTIFIED (shared list state, cache, optimistic writes)   §8
    ↓ via
Domain data-access lib (endpoint functions + mappers + query keys)              ← the ONLY HttpClient consumer
    ↓ via
platform/http (interceptors: auth → context → correlation → error → retry/timeout)
    ↓
Backend API (authoritative validation & authorization)
```

**MUST NOT**: components/features import `HttpClient` or `@angular/common/http` (lint bans it in non-data-access tagged projects). **MUST**: every endpoint function lives in a domain `data-access` lib and returns typed results (`Observable` or signal-query).

### 7.2 Contract-first typing

Backend publishes OpenAPI schema (`drf-spectacular`). Pipeline:

```text
CI job: fetch schema → openapi-typescript → libs/shared/api-types/<version>.d.ts
        └→ oazapfts/orval thin clients OPTIONAL per team preference
CI gate: schema diff vs last release — breaking changes must be labelled + approved (contract review)
```

- Generated types are **the DTOs**; domain view-models are mapped from them (§7.3). Hand-written duplicated DTO interfaces are a red flag (H).
- Types regenerated as part of affected PRs touching `api-types`; drift commits blocked.

### 7.3 Mappers — when they earn their keep

| Situation | Mapper? |
|---|---|
| API money = `{amountMinor, currency}` but UI needs formatted grouping decisions | ✅ yes — one shared money mapper |
| Dates as ISO strings; UI works with typed Date/locale strings | ✅ trivial, centralized in utils |
| Status enums stable across backend+frontend | ❌ no mapper — alias re-export through api lib |
| Field-for-field identical shapes for internal admin CRUD | ❌ no mapper — pass DTO through |

Rule: a mapper that copies fields without transforming meaning is ceremony; delete it. Every justified mapper is tested at integration level (§13).

### 7.4 List contracts (pagination · filtering · sorting · search)

```ts
interface PageRequest  { page?: number; size?: number }
interface PageResponse<T> { items: T[]; page: number; size: number; total: number }
// Query params convention:
//   ?page=1&size=50&sort=-createdAt,status
//   filters:  f[status]=CONFIRMED&f[branch]=12      OR repeated params if API prefers
//   search:   q=<raw>&qFields=code,name
```
Domain queries extend these primitives into their own typed filter interfaces (`OrderQuery extends PageRequest { status?: OrderStatus[] }`) built by small pure "query builder" functions — testable, reusable by dashboards/reports exports.

### 7.5 Retry · timeout · deduplication policy (centralized in platform/http)

| Concern | Rule |
|---|---|
| Timeout | default `requestTimeoutMs` (30s); uploads/downloads exempt; aborts map to normalized timeout error |
| Retry | GET/HEAD on network error or 502/503/504: max 2 attempts, exponential + jitter. Mutations never auto-retried (duplicate financial postings!). Idempotency-key header MAY enable safe mutation retries server-side |
| Dedup | identical in-flight reference-data loads share one observable (`shareReplay(1)` with TTL via `RefCountedCache` util). Dashboards hitting same endpoints reuse cache by key |
| Cancellation | every list/search request carries an `AbortSignal` bound to component/route lifetime; route leave cancels stragglers |
| Race conditions | search pattern §7.7 guarantees latest-wins |

### 7.6 Error normalization

One shape, produced by exactly one interceptor:

```ts
class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,            // stable machine code e.g. 'validation.failed', 'fiscal.period.closed'
    readonly messageKey: string,      // i18n key resolved centrally; never raw server English text to users
    readonly fieldErrors?: Record<string, string>,
    readonly correlationId?: string,  // shown in UI for support escalation
    readonly details?: unknown,
  ) {}
}
```

Presentation policy lives centrally too — features never write toast wiring per endpoint:

| Error class | UX behavior |
|---|---|
| validation w/ fieldErrors | inline form errors; summary toast optional |
| domain rule violation (`code` matches known set) | warning toast + highlighted target element |
| auth/session | session-expired flow (§6.3), no error toasts |
| permissions 403 | dedicated "no access" panel with grant-hint |
| network/server | red toast + retry action when idempotent |
| unexpected | toast (correlationId visible) + telemetry capture §14 |

Domain-specific *interpretations* (e.g., credit-block codes mapping to a drawer showing customer limit) attach via typed registry extensions owned by the domain — extension points exist; infrastructure does not special-case business strings.

### 7.7 Search/list standard (kills race conditions & duplication)

Reusable store recipe (`platform/http` exports it; domains instantiate typed):

```ts
const ordersList = createListStore({
  queryBuilder: q => buildOrderQuery(q),          // pure, testable
  fetcher:     (req) => ordersApi.getOrders(req), // single source
  pageMode:    'server',                          // mandated ≥ threshold rows (§12.3)
});
// internals: debounced signal input (250ms), switchMap-like cancellation via
// rxResource/request-version token, stale-response guard, ctxVersion in cache key.
```

Every searchable screen uses this; none hand-roll debounce+subscribe pairs.

### 7.8 Optimistic updates — policy

Allowed **only** when ALL hold: entity field low-risk (labels/priorities, not amounts/statuses), operation idempotent, conflict detection available (ETag/version), rollback path implemented, and the affected aggregate's store registered invalidation on failure. Financial postings, stock movements, approvals → **always await server confirmation**; UI shows pending state instead. Concurrency conflicts surface as reload-with-merge dialog reading server state (§11.5).

### 7.9 File & long-running endpoints

Files: upload progress via `reportProgress:true`, streamed downloads with filename sanitation, presigned preview. Long-running ops use platform `operations` client: `POST /ops/x` → `{operationId}` → poll status (backoff 0.5s→2s→5s cap) or SSE push bridged to notification center (§11.6).

### 7.10 API versioning stance

Frontend targets `/api/v1`. Breaking versions: new major path released only after compatibility window; generated types pinned to deployed major per release train. Non-breaking additions may be consumed immediately; removals tracked via deprecation headers surfaced in dev-mode console warnings and CI report.

## 8. State Management Rules

### 8.1 The seven state categories and their default homes

| State category | Examples | Default home | Shared via |
|---|---|---|---|
| **Local component** | selected tab, collapsed panel, hover | plain field / `signal()` in component | never — template only |
| **Form state** | draft order editor values + validation | Typed `FormGroup`/`SignalForms` instance; drafts preserved per dirty-guard (§11.2) | parent↔child within feature lib |
| **URL/query-param** | list page, sort, filters, selected row, detail entity id | the URL itself (router = source of truth) | routerLink + query binding |
| **Server (cached)** | order lists, customer records, reference data | domain data-access store (SignalStore) with context-keyed invalidation | public selectors from api/domain-access surface |
| **Feature-shared** | multi-pane screens (list+detail), wizard steps across components | one SignalStore **inside that feature/domain**, named per aggregate | facade of selectors + command methods |
| **Application/session** | identity, grants, workspace contexts, language | platform stores (`auth`, `permissions`, `contexts`, `i18n`) | their public APIs |
| **Derived/ephemeral UI cache** | computed totals from form lines | `computed()` chain — not stored anywhere | component tree |

### 8.2 Decision ladder — walk DOWN until you must stop

1. Can a single component own it? → local signal.
2. Is it encoded navigation intent? → URL, full stop.
3. Is it purely server-derived for one route? → `rxResource`/`resource` locally with cancellation; no store.
4. Do ≥2 sibling features/screens share a dataset & mutations? → typed SignalStore **co-located in the owning domain's data-access layer**.
5. Does it describe *who/where I am* rather than *what I look at*? → platform store (one of four above).
6. Nothing else fits and >3 domains truly need identical shared mutable state → raise an architecture review; global stores beyond platform are born rarely and formally.

### 8.3 Signals alone vs SignalStore

| Signals alone enough | SignalStore justified |
|---|---|
| single-component state; simple derived views | multi-component consumers + commands + invalidation semantics |
| ephemeral UI choices | aggregates with lifecycle (loaded/editing/saving/error) |
| pass-through presentational props | need devtools inspection, entities with ids & partial updates |

Sanctioned tool: **@ngrx/signals SignalStore** specifically (method-based, tree-shakable, no action classes). Classic NgRx Store/Effects is **not** adopted: its ceremony buys nothing under signals-based DI and adds indirection that hurts DX at review time. Akita/MobX/etc: rejected (§16).

### 8.4 Server-state discipline

- One request-per-key pattern: every fetch goes through store keyed by `(ctxVersion, endpointParams)`; stale-guard rejects out-of-order responses (pairs with §7.7 list standard).
- Refetch triggers are explicit and enumerable: initial load · context version bump (§5.4) · targeted push event (§5.2) · user-initiated refresh. No polling loops by habit; interval polling needs config-owned justification.
- TTL-based soft caches exist only for genuinely static reference data (currencies, units-of-measure, fiscal calendars) inside a dedicated ref-data store.

### 8.5 Effects discipline

- Prefer `computed()` over effects; effects sync to **external sinks** (router nav after success, analytics beep, websocket join) — never effect→effect chains and never state→state loops.
- Every effect carries teardown on destroy scope (`DestroyRef`/`takeUntilDestroyed`) and correlation logging. Untracked writes to signals inside effects require justification comment reviewed in PR.

### 8.6 Avoiding god stores & duplicated truth

- **Naming test:** if the store's name says more than two domain nouns ("OrderWorkflowAndCustomerAndNotifications…"), split it. Stores map ~1:1 to aggregates or routes-group use cases.
- **One source rule:** for each fact, one owner. e.g., "current company" owned by contexts store; a sales screen reading it uses the selector, never copies value into local field at init.
- **URL round-trip test:** deep-linking into any list/detail restores *the same screen* without store replay — proves URL owns navigational state, stores own volatile caches.
- Platform stores expose read-only signals; mutation methods live beside them, giving a single audit point (log line at each transition §14.1).

## 9. Nx Enforcement: Making Boundaries Real

> **Goal:** a developer who writes a forbidden import gets a red screen locally in seconds — not an email from an architect in three weeks. Prevention over persuasion.

### 9.1 Tagging scheme

Every project carries exactly one `scope` + one `type` (+ `visibility` where relevant):

| Tag family | Values | Example |
|---|---|---|
| `scope:*` | `platform`, `design-system`, `shared`, or `domain:<name>` (`domain:sales`) | `"scope:platform"` |
| `type:*` | `app`, `config`, `auth-infra?` — concretely: `feature`, `data-access`, `ui`, `domain-model`, `util`, `infra` | `"type:data-access"` |
| `visibility:*` | `public` (only domain api libs) | `"visibility:public"` |

```jsonc
// libs/domains/sales/data-access/project.json  →
{ "tags": ["scope:domain:sales", "type:data-access"] }
// libs/domains/sales/api        →
{ "tags": ["scope:domain:sales", "type:data-access", "visibility:public"] }
```

### 9.2 Module-boundary constraints

Nx evaluates depConstraints top-down, first match wins; specific entries precede the catch-all. When adding a domain, run the scaffold script which appends its constraint rows — boundaries file is generated, never hand-edited.

```jsonc
// nx.json  → "boundary": { "enforceModuleBoundaries": true, "depConstraints": [...] }
[
  // apps orchestrate everything through published surfaces only
  { "sourceTag": "type:app", "onlyDependOnLibsWithTags": ["visibility:public", "scope:design-system", "scope:platform", "scope:shared"] },

  // platform is terminal: never reaches business scopes upward except nothing — domains depend on platform, reverse forbidden
  { "sourceTag": "scope:platform", "onlyDependOnLibsWithTags": ["scope:platform", "scope:shared", "type:util"] },

  { "sourceTag": "scope:design-system", "onlyDependOnLibsWithTags": ["scope:design-system", "scope:shared", "scope:platform"] },
  { "sourceTag": "scope:shared", "onlyDependOnLibsWithTags": ["scope:shared", "type:util"] },   // shared must stay leaf-ish

  // domain internals: same-domain siblings + everything public/domain-free
  { "sourceTag": "scope:domain:sales", "onlyDependOnLibsWithTags":
      ["visibility:public", "scope:platform", "scope:design-system", "scope:shared", "scope:domain:sales"] },

  // ⛔ catch-all AFTER specifics: any other scope may consume only public/design/platform/shared leaves
  { "sourceTag": "*", "onlyDependOnLibsWithTags": ["visibility:public", "scope:platform", "scope:design-system", "scope:shared"] }
]
```

Result guarantees:
- sales `data-access` importing inventory `_internal/…` → blocked (inventory internals untagged-public).
- accounting importing `domains/purchasing/api` → allowed via `visibility:public`.
- design-system reaching into `domains/*` → blocked by own row.
- new lib missing tags → fails: unknown-scope default deny.

### 9.3 Type-layer bans (ESLint matrix)

Beyond project-level tags, directory-scoped `no-restricted-imports` kills API misuse regardless of project:

```jsonc
// root eslint flat-config fragment
{
  files: ['libs/domains/**/feature-*/**', 'libs/domains/**/ui/**'],
  rules: { 'no-restricted-imports': ['error',
    { paths: [{ name: '@angular/common/http' }] }                       // no HttpClient outside data-access
  ]},
},
{
  files: ['!libs/platform/config/**'],                                  // everywhere else:
  rules: { 'no-restricted-imports': ['error',
    { patterns: ['*environments*', 'assets/runtime-config*' ]}          // config through APP_CONFIG only
  ]},
},
{
  files: ['libs/domains/**/*.component.ts'],
  rules: { 'no-restricted-syntax': ['error',
    { selector: 'CallExpression[callee.name=/bypassSecurityTrust/]'}    // sanitizer bypass = flagged
  ]},
}
```

Plus barrel hygiene: `import/no-restricted-paths` or Nx's enforce on deep imports keeps `@erp/sales/domain/src/lib/_internal/x` unreachable.

### 9.4 Public API surface checks

CI job runs `nx graph` and asserts: every dependency edge entering a `visibility:public` lib comes through its index exports (manifest validated against generated API report akin to `api-extractor`). Growing export list diff appears in PR description automatically.

### 9.5 Ownership & review gates

`CODEOWNERS` maps `libs/domains/sales/**` → @team-sales etc.; platform dirs → @team-platform with two-reviewer rule for platform changes (blast radius). Feature PR touching another team's api lib auto-requests their review.

### 9.6 Affected-based CI pipeline contract

```text
pr:      nx affected -t lint test build          (parallel shards; cache hit expected on untouched)
main:    nx affected build; e2e smoke suite; release artifacts tagged immutably
nightly: full nx run-many (catches what aff-partition drift hides), license/vuln audit, tag-drift check,
         translation-key coverage, bundle-budget report
```

Cache & distribution: remote Nx cache shared team-wide from day one. Target rule: `lint+affected-build < 15 min` at 100 projects.

### 9.7 Scaffold discipline

`npm run g:domain -- --name=hrm` generates: folder set §3.1, tags, depConstraints rows, route/nav registration points, translations scope scaffold, checklist template link. Domains are born compliant, not retrofitted.

## 10. Design System

### 10.1 Relationship map

```text
Angular CDK            → headless behavior primitives (overlay, virtual scroll, drag, a11y)
Angular Material       → themed generic components (inputs, menus, tabs…)
ERP design-system lib  → tokens, wrapped branded surfaces, composed ERP widgets
Feature UI             → feature-specific composition of the above
```

### 10.2 Wrap vs pass-through — the rule

| Component class | Decision |
|---|---|
| Simple inputs/buttons/checkbox with defaults | **pass-through** Material component via theme (no wrapper); features import directly from design-system barrel only |
| Components needing enforced ERP look+behavior across all screens (data table shell, page header, action bar, status badge/timeline, attachments panel, empty-state, date-range picker w/ fiscal presets) | **wrapped** in design-system once; direct raw usage of underlying pieces banned where substitution would break consistency |
| Fully bespoke business widgets (order-lines matrix editor) | **project-owned** inside owning domain `ui` — not the design system's job |

Rationale: wrapping *everything* re-implements Material and slows upgrades to zero; wrapping *nothing* lets 300 screens drift into 12 dialects. The split above keeps wrap-count low but consistent-critical high.

### 10.3 Tokens & theming

Material M3 token system as the single styling substrate: color roles, typography scale, spacing/density, elevation mapped from brand palette. SCSS surfaces token variables; no hardcoded hex in feature styles (**lint: style-declaration color scan**). Dark mode & density variants derive from token overrides only.

### 10.4 Consistency mechanics at 300 screens

- Grids always through `<erp-table>` shell (column defs, sticky header, virtualization threshold, pagination bar, saved views hook, empty/loading/error slots).
- Forms via standard field wrappers (`erp-field` = label + hint + error display from ApiError.fieldErrors), typed reactive forms.
- Dialog/toast only via platform/dialogs & notifications services.
- Review checklist section "UI" (Artifact G) is the enforcement at PR time; visual regression snapshots on wrapped components catch drift automatically (§13.6).

### 10.5 RTL & accessibility centrally

- Root `dir` switch only (§5.5); components consume logical properties exclusively; Material handles mirroring internally.
- Wrapped components guarantee keyboard paths & ARIA semantics once — features inherit. Focus trap on dialogs, grid keyboard nav spec'd in one place, axe-core checks run against design-system stories in CI.
- Translation placeholders preserve meaning per script (e.g., date order preference ar/fr/en differs — formatter wrappers own that, never templates).

## 11. ERP-Specific Architecture Concerns

Each pattern below is broadly applicable to operational enterprise software (documents, approvals, ledgers) — no invented business rules, only architecture.

### 11.1 Contexts on screen — the standard UX contract

Context switcher in the app chrome shows active company (+branch/warehouse/fiscal when relevant); switching runs the guarded workflow of §5.4. Every list/read model displays its scope implicitly through the chrome badge, so screens never repeat company pickers per page (an early-stage smell that forks into inconsistent scoping bugs). Deep links carry `?co=…&fy=…` params restoring scope after navigation/reload; illegal contexts resolve to selector wizard rather than error walls.

### 11.2 Document lifecycle handling

Operational documents (orders, invoices, transfers…) share one structural approach:

```text
domain lib:   statusMachine = { states: [...], events, transitions, guards }
api:          exposes canTransition(doc, event)  ← pure domain rule fn (tested!)
ui shared:    <erp-status-flow> renders progress/timeline/actions from machine meta
feature:      action buttons render only permitted transitions ∩ user grants
data-access:  command endpoints return new state snapshot → store patch
```

- Transition commands are **server-confirmed**; optimistic transitions never (money moves).
- Guard failures surface with actionable reason (`code: 'fiscal.period.closed'` → toast + link to fiscal setup screen).
- Dirty-editor guard blocks accidental navigation/abandonment with save/discard dialog; draft preservation covers session expiry (§5.6).

### 11.3 Financial actions pattern

Any action that posts amounts, locks periods, approves spend, or generates legal docs follows **confirm-with-evidence**: platform `dialogs` provides `financialConfirm({summary rows, totals, warnings[], reasonInput?, secondApprover?})`. Reason capture becomes part of audit trail metadata sent with the command. Buttons show pending state until server ACK (§7.8). Timestamps displayed via tenant timezone formatter (§11.8).

### 11.4 Audit-sensitive operations

Every mutating command UI must display last-modified by/at and expected version → interceptor sends `If-Match: v42` or body field; API responds 409 on conflict (§11.5). Frontend never "helpfully" retries writes that failed on conflict/logical-constraint; those flow to the conflict UX instead. Audit log viewer (admin feature) reads server-side trail; the SPA adds nothing client-side except correlation ID surfacing.

### 11.5 Concurrency & conflicting edits

Standard conflict resolution = reload-and-inspect:

```text
409 received → conflict dialog (yours vs current diff summary by section)
             → "Reload my form" | "Discard mine" (never silent overwrite)
```
No distributed lock UIs (editing banners) for MVP scope; doc-level version stamping is sufficient at typical ERP concurrency rates and avoids a fragile presence system. MAY add optimistic presence indicators later per aggregate if measured pain exists.

### 11.6 Long-running operations & background jobs

```text
submit: POST /ops/import → { operationId }          (202 semantics)
track:  platform/operations store keyed by operationId
        · polling backoff 0.5→2→5s (cap), jittered
        · SSE bridge pushes {state, progress, resultRef?} → skips polls
resume: active operations restore on reload via GET /ops?mine=active
done:   notification center entry + targeted cache invalidation (§5.2)
```
UI surfaces: navbar running-jobs indicator; ops detail drawer with cancel (when supported). Errors map to same ApiError normalization so failed imports present downloadable logs like other files (§5.3).

### 11.7 Real-time notifications ↔ data freshness

Single stream service; domains wire *targeted* invalidation adapters:
```ts
on('order.confirmed', e => ordersStore.invalidate(e.orderId)); // typed envelope
```
Unbounded growing signal lists are forbidden — notification center keeps windowed store (last N + server pagination drawer).

### 11.8 Money · time · number formatting rules

| Concern | Rule |
|---|---|
| Storage/transport | ISO timestamps UTC; amounts as `{minorUnits, currency}` |
| Display | sole path: Intl wrappers (`fmtMoney`, `fmtDate`, `fmtQty`) honoring locale + config currency choice; template-side `Number(...)` formatting banned |
| Timezone | tenant timezone from profile for dates; absolute instant shown in tooltips for auditors ("14:05 UTC+1") |
| Multi-currency columns | display currency ≠ base currency explicitly marked in column header, conversions never implicit |

### 11.9 Large tables · reporting · dashboards · search

- Grid mandate thresholds (§12.3): >500 rows total → server paging mandatory; >200 concurrently rendered → virtualization on.
- Exports run server-side as ops jobs (§11.6) with file delivery through notifications; client-side CSV of paginated slices is an anti-pattern that lies about totals.
- Report printing = dedicated print routes with print stylesheet + stable pagination (browser print of grids is not a report).
- Dashboard widgets load lazily via defer/viewport triggers, each widget = independent query w/ dedup via ref-data caches; widget registry is composition-over-plugin: plain components listed in a const array, not reflection magic.
- Advanced filtering: reusable `<erp-filter-builder>` appears only once ≥2 domains need it; before that each domain ships simpler typed chips — premature generic query-builders rot quickly (§15).

## 12. Performance Defaults

Philosophy: **architectural defaults first, opportunistic tuning second.** The defaults below ship with the skeleton so slow patterns cannot become endemic; anything beyond is measured (Web Vitals + §14 timings) before engineered.

### 12.1 Build-level

| Mechanism | Default |
|---|---|
| Nx affected on all PR gates | yes (§9.6) + shared remote cache day one |
| Lazy loading | every routed feature lib lazy via route-level `loadChildren`/`import()` |
| Preloading | selective: shell nav primaries preloaded after idle (`PreloadAllModules` variant w/ delay), heavy domains only on demand |
| Code splitting | defer blocks for below-fold heavy sections (charts, editors); per-domain chunks fall out naturally from routing |
| Dependency discipline | new dep requires bundle-impact note on PR (added KB to initial vs lazy chunk); initial-budget gate enforces cumulatively |
| Budgets (initial) | warn 900 kB · error 1.2 MB raw (≈250–300 kB gz); any component chunk warn 400 kB |
| Angular builder | esbuild-based `@angular/build` application builder everywhere; no dual webpack configs |

### 12.2 Runtime rendering baseline

- Zoneless-first configuration (`provideZonelessChangeDetection`) on current majors; OnPush explicit during the migration window — either way components are signals-driven.
- Template hygiene: pure `computed()` for derived values; no method calls returning fresh objects per CD pass; native control flow `@for (track expr)` mandatory.
- Subscription hygiene automatic through signal APIs/rxResource; manual subscribe requires teardown pairing (lint-family review item).

### 12.3 Large-data defaults (the actual ERP killers)

| Scenario | Architectural default |
|---|---|
| Any list expected >500 rows | server-side paging/sort/filter mandatory (§7.7 `pageMode:'server'`) |
| Rendered rows >200 visible | CDK virtual scrolling inside `<erp-table>` shell |
| Page size default 50, max selectable 200 | prevents "download the ledger" grids |
| Forms with >150 controls | wizard/segmented sections rendered by section state, not one mega-form node |
| Dashboards | widgets viewport-deferred; each ≤1 primary query; aggregations server-side |
| Typing performance in big inputs/forms | uncontrolled value writes throttled through `linkedSignal`/marks rather than full recompute chains |

### 12.4 API traffic containment

Dedup of reference data (§7.5), cancellation on navigation (§7.7), batch endpoints where backend offers them for multi-widget screens, conditional GETs accepted transparently by stores (ETag reuse), SSE replacing aggressive polling.

### 12.5 What is *not* done prematurely

No hand-rolled memoization layers, no workers before measured CPU pain, no IndexedDB persistence layer until an offline requirement exists (rare behind-auth ERP), no virtual scroll on every small table. Each forbidden-by-default optimization returns when a number justifies it.

## 13. Testing Architecture

### 13.1 The pyramid, by *responsibility* — not by tool count

| Level | What it verifies | Where it lives | Runner |
|---|---|---|---|
| **Unit** | pure business logic: domain rules (`canConfirm`, tax/rounding), mappers w/ transformations, query builders, form validators | beside code in each lib | Vitest |
| **Store/data-access integration** | store lifecycle incl. context-keyed invalidation, retry/error normalization contracts vs mocked server (MSW), header propagation | data-access libs + platform/http | Vitest + MSW |
| **Component** | tricky interactive pieces only: grid shell behaviors (sort/filter keys, selection sync to URL), field wrapper error display, status-flow actions ∩ grants | design-system + complex widgets | Vitest + Angular Testing Library |
| **E2E golden paths** | real workflows against seeded staging: order→confirm→invoice→payment run; company switch mid-flow; 401 renewal during editor session; permission-denied screens behave; ar/fr/en+RTL spot checks | `apps/e2e` | Playwright |

### 13.2 Mandatory coverage rules (risk-based, not %)

- Every pure rule fn from `domain/*`: unit-tested including boundary cases (closed fiscal period, zero-qty lines, currency mismatch).
- Permission matrix: for each released feature, a table test asserting visibility/dispatch behavior per grant-combo fixture.
- Financial confirm flows: E2E at minimum happy-path + conflict-reload path once per aggregate family.
- Error normalization contract: platform suite covers every ApiError class → expected UX dispatch; violations fail CI.
- i18n key existence + ICU placeholder integrity across all locales: nightly job (§9.6).

### 13.3 Explicitly discouraged tests

Snapshot-the-whole-template tests, "did this method get called" spies on private collaborators, tests asserting selector names/CSS classes of pass-through Material components, param-permutation spam generated to game coverage dashboards. These rot faster than they pay.

### 13.4 Test doubles policy

Real domain types over mocks-of-your-own-types; backend fakes via MSW handlers (typed by generated OpenAPI) so integration tests exercise the mapper/interceptor reality. Component stubs are plain lambdas unless overriding input/output matters.

### 13.5 Speed contract

Unit suites < 10 s project-local (<90 s full affected), no real network anywhere below E2E, parallel shards in CI. Tests that get slow get quarantined with an owner, not tolerated into flakiness.

### 13.6 Visual regression (SHOULD)

Playwright screenshot diffs on wrapped design-system components in LTR+RTL × light/dark matrix; baseline updates require design-owner review. Prevents the classic 40-screen drift of paddings and states.

## 14. Observability

### 14.1 Logging

One structured logger (`platform/logging`):

```ts
log.info('orders.confirm.ok', { orderId, totalMinor, tookMs });
// emitted shape: { ts, level, msg, corrId, userId?, companyId, module:'sales', details }
```

- Levels gated by `obs.logLevel`; remote sink (§14.4) receives warn+ always, debug/info sampled per `sampleRate`.
- **Redaction pipeline runs before any sink**: token headers, full payloads containing PII/amount arrays (summarize instead), authorization blobs are stripped by filter chain; new filters added when new sensitive field classes appear.
- Console in dev is an inspector of the same events — never a parallel log path.

### 14.2 Global error capture

Angular `ErrorHandler` + `window.onerror` + `unhandledrejection` all funnel to one reporter: fingerprint (message-normalized → grouping), user context snapshot (route, grants-version, context ids), breadcrumb tail from logger buffer, then telemetry send. Errors shown to users carry the correlation id (§7.6) closing the support loop.

### 14.3 Correlation spine (browser ↔ backend ↔ jobs)

```text
mint uuid at action start (route change / dialog open)      [telemetry store]
    → X-Correlation-Id on every HTTP call (reused if server set one first)
    → echoed by API into its logs & worker job metadata
    → SSE/WSS events carry it back
support view: "give me your reference code" matches exactly one browser timeline
```
Rules: IDs are opaque UUIDs (no PII), rotated per logical action not per request, injected uniformly via interceptor so no feature can forget it.

### 14.4 Sinks & provider adapters

Default: self-hosted JSON collector endpoint (config-owned URL) writing to existing log infra. MAY adopt Sentry (self-hosted or approved cloud) for error aggregation ergonomics — swap happens inside `platform/telemetry`, zero feature changes.

### 14.5 Performance monitoring

Web Vitals reported post-load-sample; custom timing marks at architecture points only: boot phases (config→auth→shell), route chunk load-to-render, grid first-page render, save round-trips (per aggregate class). Dashboard aggregates these by release tag for regression alerts — this replaces "it feels slower" archaeology with numbers.

## 15. Maintainability & Abstraction Policy

> **The single test:** introduce an abstraction because it protects a boundary or removes meaningful duplication — never because abstraction itself looks architectural.

### 15.1 The kill-list

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| God service ("DataManager", "AppService") | one change → workspace-wide blast radius; untestable | split by concern into platform/domain libs above |
| God store | invalidation coupling across unrelated features | §8.6 naming test |
| God component (600-line screens w/ logic in templates) | templates become business-rule locations (H red flag) | page = composition + signals; rules live in domain |
| Generic *Manager/*Helper/*Util utility namespaces | orphan functions nobody owns | named topic utils w/ admission gate §15.3 |
| Service locator / global injector scans | DI-by-magic breaks treeshaking & reasoning | explicit inject() paths |
| Premature facades over one method | two indirections for zero protection | direct use until second consumer exists (then decide which side owns abstraction) |
| Duplicate abstractions per domain (three upload services) | drift + triple bug surface | platform surface + typed extension point §5.3 |

### 15.2 Abstraction admission checklist

A new shared abstraction enters only when it passes ALL:
1. ≥2 concrete consumers exist today (not hypothetical).
2. It removes real duplication measured in ≥40 LOC-equivalents or a correctness risk (security/money).
3. A boundary actually needs protecting (else duplication was fine).
4. One owner declared; docs include withdrawal criteria.
5. Extension point reviewed against "does this force consumers into template-method cages?" — prefer composition params.

### 15.3 `shared/` admission (repeat-worth anchoring)

PRs adding to `shared/ui` or `shared/utils` must link the two consumer sites, declare API, add usage doc line, and mark ownership. Quarterly sweep deletes anything unused by >0 imports (CI report makes this cheap). The folder stays under ~30 projects forever; growth beyond means domains are under-modeled.

### 15.4 Code-size hygiene

- Feature libs average ≤ ~1.5k LOC at maturity; data-access stores ≤ ~400; components single-digit files each.
- Barrels re-audited when import graphs grow: exports removed as often as added.
- Dead code: Nx unused-project/file reports monthly; deletion PRs are celebrated, not feared (tests make removal safe).

## 16. Library Stack Review

Scoring dimensions applied: problem solved · alternative cost · bundle/build impact · lock-in risk · stability for production long-haul. Verdicts:

### 16.1 Final stack (keep small and intentional)

| Library | Problem solved | Alternatives considered → why rejected | Lock-in/bundle | Verdict |
|---|---|---|---|---|
| Angular (latest stable majors adopted every other major within one quarter) | framework itself | migration frameworks → far larger rewrite cost later | intrinsic | MUST |
| @angular/material + @angular/cdk | accessible component baseline, behavior primitives | PrimeNG/Clarity → wrap-cost similar but M3 tokens + CDK depth win; custom-from-zero → a11y years lost | high but wrapped-boundary-controlled | MUST |
| @ngrx/signals (SignalStore only) | shared state devtools/lifecycle discipline | hand-rolled store → subtle cache bugs multiply per team; classic NgRx → ceremony rejected §8.3; Akita/MobX-XState etc. → smaller ecosystems/different paradigm | tiny (tree-shaken) | SHOULD→MUST once ≥2 shared stores |
| zod | config schema validation w/ great errors | valibot (smaller) acceptable alt; ajv (JSON-Schema world) heavier DX; none → prod misconfig caught too late | few KB parse-path | MUST |
| transloco (+ locale plugin) | runtime multi-lang + lazy scopes + ICU | @angular/localize → per-locale builds conflict w/ build-once; ngx-translate → aging core, weaker scope ergonomics | runtime loaders keep base lean | MUST (ar/fr/en requirement) |
| openapi-typescript | contract types from OpenAPI | orval/ng-openapi-gen codegen clients → more moving parts than thin per-domain fns; hand DTOs → drift rot proven at scale | types only, zero runtime | MUST |
| angular-auth-oidc-client (or oidc-client-ts directly) | OIDC-PKCE lifecycle done right | manual token mgmt → renew/race bugs are security incidents | moderate, tree-shakable-ish | MUST (choose ONE) |
| Playwright | E2E golden paths + visual regression | Cypress → license/multi-tab/arch limits vs ERP headless CI needs | dev-only | MUST |
| Vitest + @testing-library/angular | fast unit/component testing | Jest/Karma → slower/vestigial | dev-only | MUST |
| MSW | network-layer fakes exercising real stack | handwritten interceptors/spies → bypass interceptors (the thing under test) | dev-only | MUST |
| ESLint + @nx/eslint-plugin boundaries + custom restrictions | enforcement layer P1 | review-only culture → decay documented RF-02 | dev-only | MUST |

### 16.2 Deferred-on-purpose (MAY, evidence-gated)

| Candidate | Gate before adoption |
|---|---|
| ECharts (or similar) | dashboards milestone reached; lazy-chunk loaded; budget check passes |
| AG Grid Enterprise | measured need beyond erp-table shell capabilities (grouping/filter perf at 100k rows) + commercial license accepted |
| date-fns/dayjs | only when timezone arithmetic beyond Intl wrappers proved necessary |
| TS-pattern | when status machines get hard to read in switch blocks |
| RxJS operator suites/custom things | whenever plain fns stop being clearer |

### 16.3 Rejected outright (with reasons)

Redux-style global-store-everywhere, event-bus libraries, decorator-based container/framework hybrids, UI kit multipacks alongside Material, lodash-fulltree (import per-fn or drop), moment.js (legacy tz model), micro-frontend tooling absent org driver (§2.1), custom reactive form engines competing with Angular forms.

### 16.4 Upgrade policy

Adopt latest Angular major every other major (≈6-month cadence trailing); run `nx migrate` batches per domain boundary; design-system wrap-count low precisely to keep this cheap. Deprecations surfaced by nightly build with deadline tracking — long-lived deployments die of deferred upgrades otherwise.

---

# Part III — Reference Artifacts

## C. Final Folder Structure

```text
erp-workspace/
├─ apps/
│  ├─ erp-shell/                          # single SPA entry: main.ts (config→bootstrap), app.routes.ts,
│  │   └─ src/app/layout/                 # chrome: context switcher, nav, notification center, job drawer
│  └─ erp-e2e/                            # Playwright suites + fixtures + seeded-data harness
│
├─ libs/
│  ├─ platform/
│  │  ├─ config/                          # RuntimeSchema(zod) · loadRuntimeConfig() · APP_CONFIG · grouped tokens
│  │  ├─ http/                            # interceptors(auth,ctx,corr,err,retry/timeout) · ApiError · createListStore
│  │  ├─ auth/                            # OIDC-PKCE lifecycle · refresh mutex · identity signal · logout sweep
│  │  ├─ permissions/                     # grants store · has()/hasAny()/hasAll() · *erpPerm · canMatchPermissions
│  │  ├─ contexts/                        # company/branch/warehouse/fiscal/currency stores · ctxVersion · switch guard
│  │  ├─ i18n/                            # transloco root · LocaleService(lang→dir→locale) · fmtMoney/fmtDate/fmtQty
│  │  ├─ notifications/                   # toast service · SSE/WSS stream svc · typed event registry · center API
│  │  ├─ dialogs/                         # confirm/prompt/financialConfirm/workflow-dialog standards
│  │  ├─ files/                           # upload/download/presign client · <erp-attachments> backing API
│  │  ├─ operations/                      # long-running op client: submit/track/resume/cancel patterns
│  │  ├─ logging/                         # leveled logger · redaction filter chain · ErrorHandler integration
│  │  ├─ telemetry/                       # correlation minting · Web Vitals · timing marks · sink adapter SPI
│  │  └─ feature-flags/                   # isOn() signals source (config defaults + remote overrides)
│  │
│  ├─ design-system/
│  │  ├─ tokens/                          # M3 token mapping · SCSS variables · density/dark overrides
│  │  └─ components/                      # erp-table shell · erp-field · erp-page-header · erp-action-bar
│  │                                      # erp-status-badge/flow · erp-attachments · erp-empty-state · date-range fiscal presets
│  │
│  ├─ shared/
│  │  ├─ ui/                              # admitted-only generic widgets (capped ~15 projects)
│  │  ├─ utils/                           # money · time · fp-lite · format guards (admitted-only)
│  │  ├─ testing/                         # platform fakes · grant fixtures · MSW handler packs
│  │  └─ api-types/                       # GENERATED OpenAPI DTO types (CI-owned; do not hand-edit)
│  │
│  └─ domains/
│     ├─ sales/
│     │  ├─ api/              → public index: SalesOrdersApi · OrderView · SALES_ROUTES · events map
│     │  ├─ domain/           → order model · status machine · rule fns (pure)
│     │  ├─ data-access/      # orders.api.ts · order.mapper.ts · orders.store.ts (ctx-keyed)
│     │  ├─ ui/               # sales-line-editor grid · customer-mini-card (shared within sales only)
│     │  ├─ feature-orders/   # list/editor/detail pages + orders.routes.ts
│     │  └─ feature-customer-360/
│     ├─ inventory/           {api, domain, data-access, ui, feature-stock-ledger, feature-transfers, feature-cycle-counts}
│     ├─ purchasing/          {…}
│     ├─ accounting/          {api, domain, data-access, ui, feature-ledger, feature-fiscal-periods, feature-payment-runs}
│     └─ administration/      {api, domain, data-access, ui, feature-users-and-roles, feature-context-setup}
│
├─ locales/                                # ar/ fr/ en/ JSON per scope: core.json, sales.json …(i18n CI-checked)
├─ tools/scripts/                          # g:domain scaffold · boundary-constraint generator · redaction tests
├─ nx.json · eslint.config.mjs · tsconfig.base.json · proxy.conf.json · runtime-config.template.json
```

Scaffold ownership notes: `locales/*`, tags, depConstraints rows and route/nav registration are generated by `g:domain` — never hand-authored per new domain.

## D. Dependency Rules

### D.1 The permitted-edges matrix (read as row may depend on column ✅)

| ↓ from \ to → | platform | design-system | shared | own domain api | own domain internal* | other domain api | other domain internal |
|---|---|---|---|---|---|---|---|
| **apps** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| **platform internals** | ✅ | ❌¹ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **design-system** | ✅ | — | ✅ | ❌ | ❌ | ❌ | ❌ |
| **shared** | ❌² | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **domain api** | ✅ | ✅³ | ✅ | ✅ | ✅ | ❌⁴ | ❌ |
| **domain internal (feature/data-access/ui/domain)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (via api lib) | ❌ |
| **other-domain internals import of me** | | | | | enforcement via visibility tags → ❌ unless routed through my api lib |

¹ Exception: telemetry/design-system foundations MAY import config tokens (`ApiConfig`) — the only sanctioned upward edge, kept explicit.
² shared stays leaf-ish; it may never reach platform to avoid cyclic platform⇄shared temptation.
³ domain api may reuse design-system *types* only if strictly needed for view contracts; prefer plain TS.
⁴ Cross-domain reads go through target's api lib *only*; that api lib itself must not import a third domain's internals (no chains).

### D.2 Rule statements (enforced where noted)

1. **MUST NOT** feature/ui/component code import `@angular/common/http` — data-access layer is sole HTTP consumer *(lint)*.
2. **MUST NOT** any project outside `platform/config` import environment/config raw sources *(lint patterns `*environments*`, template-guarded)*.
3. **MUST NOT** deep-import past any barrel (`/src/lib/_internal/**` unreadable from outside its own project) *(lint no-restricted-paths + Nx boundary)*.
4. **MUST** cross-domain access use target domain's public api library surface exclusively *(Nx depConstraints: catch-all + visibility tag)*.
5. **MUST NOT** platform depend on any domain or business noun *(depConstraints platform row)*.
6. **SHOULD NOT** shared depend on design-system (keep shared renderable in isolation); violations need justification note.
7. **MUST NOT** apps depend on domain internals directly (routes compose through each domain's SALES_ROUTES-style export).
8. **MUST** add third-party dependencies through dependency-review with bundle-impact note; allowlist license check runs CI *(SCA gate)*.
9. **MUST NOT** create new top-level folders under `libs/` beyond `platform|design-system|shared|domains` without architecture review.
10. **MUST** every project carry complete tag triple; CI fails unknown-tagged projects *(tag-drift nightly)*.

## E. Centralization Rules

Each line states: what is centralized → who consumes → what stays local.

| Concern | Centralized (owner) | Stays domain-specific |
|---|---|---|
| Configuration | loader/schema/tokens (`platform/config`) | nothing below tokens — features read grouped accessors only |
| Authentication | session lifecycle & tokens (`platform/auth`) | route declarations list required grants, not mechanics |
| Permissions | grant shape+evaluation+directive (`platform/permissions`) | which codes a screen requires; interpretation of contextual denial UX |
| HTTP | interceptors/retry/timeout/error mapping/list-store recipe (`platform/http`) | endpoint functions, mappers, cache keys per aggregate |
| Errors | ApiError shape + central dispatcher + messageKey resolution | interpreting specific domain codes into screen affordances |
| Logging/telemetry | logger, redaction, correlation spine, Vitals (`platform/logging`,`/telemetry`) | semantic event names emitted by domains |
| Notifications | transport+toasts+center (`platform/notifications`) | subscription adapters reacting to typed business events |
| Dialogs | confirm/prompt/financial standards (`platform/dialogs`) | dialog content models per flow |
| Files | upload/download/presign client+panel (`platform/files`) | document metadata screens/business typing |
| Company/Branch/Warehouse/Fiscal/Currency | context stores+headers+switch workflow+ctxVersion (`platform/contexts`) | how screens react: e.g., disable posting when period closed |
| Localization | engine+LocaleService+formatters (`platform/i18n`) | translation JSON content per domain scope |
| Feature flags | loading/evaluation surface (`platform/feature-flags`) | gating decisions inside flows |
| Status machines rendering | `<erp-status-flow>` renderer + machine meta types (design-system/domain util) | transition tables & guards per document type |
| Money/time formatting | Intl wrappers (`shared/utils` + LocaleService) | column labels/header hints about currency choice |
| Concurrency/conflicts | version-header convention + conflict dialog pattern (`platform/http` + dialogs) | merge-hints text per document type |
| Long-running ops | operation client lifecycle (`platform/operations`) | progress display nuances per job kind |

## F. Library Stack (authoritative list)

Adopted core (rationale in §16.1): **Angular · @angular/material + @angular/cdk · @ngrx/signals (SignalStore) · zod · transloco · openapi-typescript · angular-auth-oidc-client (or oidc-client-ts) · Playwright · Vitest + @testing-library/angular · MSW · Nx toolchain + ESLint boundary stack**.
Evidence-gated MAY: ECharts · AG Grid Enterprise · date-fns/dayjs · TS-pattern. Rejected outright: classic NgRx Store/Effects by default, event-bus libs, MFE tooling without org driver, UI multipacks alongside Material, moment.js, lodash-fulltree.

Adding any dependency: PR must state problem, alternative rejected, bundle delta measured on affected build, exit plan (how we'd remove it).

## G. Final Implementation Checklist

Run this after **every significant feature**. PR description contains the checked list; reviewers spot-check sampled items.

### G.1 Architecture & dependencies
- [ ] Feature code lives inside its domain; no new top-level/`shared` entry without admission checklist §15.3
- [ ] No dependency outside allowed matrix D.1 (`nx graph --file=diff.md` attached when touching public APIs)
- [ ] New library got tags via scaffold; no hand-written depConstraints edits
- [ ] Third-party dep additions include problem/alt/bundle-delta/exit-plan

### G.2 Configuration
- [ ] Zero direct reads of environments/env vars/location params for settings
- [ ] Any new runtime key added to RuntimeSchema **and** `runtime-config.template.json` (schema test passed)
- [ ] No value added that belongs to the never-shippable list §4.7
- [ ] Flags carry owner + expiry

### G.3 HTTP & API contract
- [ ] Endpoint functions live in domain data-access lib; typed through generated API types (no parallel handwritten DTO)
- [ ] Mapper justified per §7.3 or removed
- [ ] List screen uses createListStore standard (server paging beyond thresholds §12.3); URL owns filters/page/sort
- [ ] Mutations pass Idempotency-Key where supported; optimistic writes only if §7.8 whitelist holds
- [ ] Errors flow through ApiError normalization; no per-endpoint toast wiring; fieldErrors wired into form display

### G.4 Security
- [ ] Route registered with permission guard factory + correct grant codes; menu/hide vs disable policy applied
- [ ] No `bypassSecurityTrust*`, no raw innerHTML with data-derived content, external links sanitized, redirects allowlisted
- [ ] Backend team confirmation recorded that every exposed endpoint enforces grants/scope server-side
- [ ] Sensitive values absent from logs/details payloads (redaction tests extended if new field class introduced)

### G.5 State
- [ ] Decision ladder applied: no store where local signal/resource suffices; no feature-owned global state
- [ ] Cache keys include ctxVersion; context switch produces fresh views (manual test w/ company switch)
- [ ] Effects only at external sinks; teardown paired everywhere

### G.6 UI / design system
- [ ] Built from design-system shells (`erp-table`, `erp-field`, dialogs/toasts services) — no private grid/dialog/upload implementations
- [ ] All strings through translation scopes incl. pluralization (ar/fr/en files complete); RTL visual check performed (logical CSS only)
- [ ] Money/date/qty rendered via Intl wrappers only; timezones handled per §11.8
- [ ] Loading/empty/error states present for every async region; a11y labels & keyboard path pass on new interactive elements

### G.7 Forms & tables specifics
- [ ] Typed reactive forms; dirty-guard registered (route leave + context switch); drafts preserved across session loss where §5.6 applies
- [ ] Document-type screens: status machine driven actions ∩ grants; financial confirm-with-evidence dialog used for posting/approval class actions
- [ ] Version stamping/If-Match sent on mutable docs; 409 → conflict-reload UX not retry
- [ ] Attachments panel reused; upload caps honored from config

### G.8 ERP context & workflow
- [ ] Deep links restore scope (co/fy params); switcher behavior verified mid-flow
- [ ] Long-running operations go through platform/operations pattern; completion lands in notification center
- [ ] Audit-sensitive actions show modified-by/at and record reason capture

### G.9 Performance
- [ ] Bundle budget report clean (initial + chunk); lazy route confirmed (no eager import leakage)
- [ ] Virtualization threshold respected; page size caps honored; heavy sections under defer
- [ ] Dedup/cache reuse verified against §7.5 defaults; no polling without configured justification

### G.10 Testing
- [ ] Pure rule functions unit-tested with boundaries; permission matrix table test added/updated
- [ ] Store integration test includes invalidation-on-context-bump case; error-normalization cases covered via MSW handlers
- [ ] Golden-path E2E added/extended for new aggregate family; visual snapshots updated for new wrapped components
- [ ] No implementation-detail tests added (§13.3 review)

### G.11 Observability
- [ ] Meaningful log events at command success/failure paths using logger (no console.*)
- [ ] New business event types registered in notifications map if relevant; correlation id appears in surfaced errors

### G.12 Maintainability
- [ ] Files below size norms §15.4; no god anything; barrels pruned of unused exports
- [ ] Docs updated: domain README one-pager touched; translation keys lint clean

### G.13 CI/CD readiness
- [ ] Affected build/test green locally via same commands as pipeline
- [ ] Migration generated for app config changes; release notes note flag rollout & rollback path
- [ ] Checklist appended literally in PR (copy/paste block) — partial checks require inline justification

## H. Architecture Red Flags

Any reviewer seeing these raises them immediately; repeated occurrences = platform task:

| Red flag | Why it's a smell |
|---|---|
| Direct environment/config file access in features | breaks build-once-deploy-anywhere; unvalidated settings; RF-04 |
| `HttpClient` usage inside components/features | bypasses interceptors: no auth/context/correlation/error normalization |
| Global god store / god service | blast radius, invalidation coupling, review bottlenecks |
| Cross-domain internal imports (past api barrel) | invisible coupling, ownership erosion |
| Random utility dumping in shared/util | orphan logic, ownership vacuum |
| Hand-copied DTO interfaces duplicating OpenAPI types | drift rot, silent contract breakage |
| Business rules in templates/method-called-in-template | untestable logic, perf hazards, duplicated across screens |
| Private grid/dialog/upload implementations per feature | N dialects × N bug surfaces; UX inconsistency |
| Optimistic updates on amounts/statuses/postings | money-safety violation; conflict chaos |
| Console.log leftovers / parallel logging path | PII leaks, noise drowning real signals |
| Subscriptions without teardown pairing | zombie callbacks updating dead stores → memory leaks, phantom toasts |
| Effects mutating other signals' state | loops, unreviewable reactive graphs |
| Polling loops without config justification | traffic storms at scale; masks SSE failures |
| New top-level folder or "misc" namespace appearing | architecture boundary decay beginning |
| Untagged projects / edited-by-hand depConstraints | enforcement blind spots multiply silently |
| Tests asserting implementation minutiae | rot-driven culture shift; coverage theater |
| localStorage holding tokens/session artifacts | XSS-exfil risk beyond accepted trade-offs |

---

# Final Recommended Architecture

The authoritative one-page summary. If a decision dispute cannot find its answer elsewhere, this section wins.

**Workspace & boundaries**
One Nx monorepo, one SPA (`erp-shell`) + E2E app. `libs/` holds exactly four top-level namespaces: `platform/` (13 domain-free infrastructure libraries), `design-system/`, `shared/` (capped, admission-gated), `domains/<name>/{api, domain, data-access, ui, feature-*}`. Cross-domain access happens exclusively through each domain's public `api` library; internals are lint-inaccessible. Tags (`scope`, `type`, `visibility`) + Nx depConstraints + directory-scoped ESLint bans make forbidden imports impossible rather than discouraged.

**Configuration**
Build once, deploy anywhere. One immutable artifact; deployments render `runtime-config.json`; `main.ts` fetches it before bootstrap; zod validates against `RuntimeSchema`; validated frozen object is provided as `APP_CONFIG`; features consume typed grouped tokens only. Direct environment reads are lint-banned outside `platform/config`. Nothing secret is ever frontend config — PKCE OIDC requires no client secret anywhere in the SPA.

**Security posture**
OIDC Authorization Code + PKCE; tokens in memory with single-flight renewal (refresh in httpOnly cookie where infra allows, never localStorage). Permissions = backend-issued grant codes evaluated centrally for UX; every authorization decision re-verified server-side; guards/hiding are affordances only. CSP, sanitizer discipline, redirect allowlist, SCA automation protect the runtime surface.

**HTTP/API**
Contract-first: OpenAPI-generated DTO types regenerated in CI with drift gates; thin typed endpoint functions per domain data-access lib; mappers only when they transform meaning. Central interceptor stack provides auth header, context headers (`X-Company-ID`…), correlation ID, timeout/retry-on-idempotent-only, and normalized `ApiError` with central UX dispatch policy. One reusable list-store recipe eliminates race conditions, request storms, and hand-rolled debounce code everywhere.

**State**
Decision ladder ends god stores before they start: local signal → URL → route-local resource → domain SignalStore keyed by context version → platform session/context stores. @ngrx/signals SignalStore is the sanctioned shared-state tool; effects only synchronize to external sinks. Server state is cached, never owned; URL is the single truth for navigational state.

**ERP operation model**
Company/branch/warehouse/fiscal/currency live as reactive platform context stores: propagated on every request, woven into cache identity via `ctxVersion`, guarded during switching, persisted per user. Documents run through shared status-machine rendering; financial actions use confirm-with-evidence dialogs; writes carry version stamps (409 → conflict-reload UX); long-running operations use the standard submit/track/resume job client bridged to a single notification stream that also powers targeted cache invalidation.

**i18n / RTL**
Runtime language switching over one artifact (ar/fr/en); Transloco scopes lazy-loaded per domain; LocaleService owns language → root `dir` → Intl formatter chain; logical CSS properties everywhere; no per-locale builds.

**Design system**
Material/CDK underneath, theme-token substrate above, low wrap-count (table shell, field wrapper, page chrome, status flow, attachments panel) so consistency survives 300 screens and upgrades stay cheap.

**Performance defaults**
Affected-based CI with remote cache; budgets enforced (initial ≤1.2 MB raw / warn 900 kB); zoneless-signals baseline; server-side paging mandated beyond 500 rows with virtualization beyond 200 rendered rows; deferred dashboards; deduped reference data; SSE instead of habitual polling.

**Testing & observability**
Risk-based pyramid (pure rules unit-tested, MSW-driven store/integration tests incl. error-normalization and context invalidation contracts, component tests only for tricky interactive shells, Playwright golden paths + RTL/light/dark visual snapshots). Structured redacted logging, global error reporting with fingerprints, Web Vitals + boot/route/save timing marks, and UUID correlation IDs threaded browser→API→workers so support questions end with one reference code.

**Delivery culture**
Rules classified MUST/SHOULD/MAY/MUST NOT with automated enforcement prioritized; 13-part feature checklist appended to PRs; red-flag catalog for reviewers; scaffolds generate compliant domains; abstractions require two real consumers plus boundary protection to exist.

---

## Closing note on evolution

This guide is versioned with the repo (`docs/architecture.md`). Amendments follow an ADR process (short docs capturing contested decisions with the §10-criterion scoring), and quarterly architecture reviews sweep red flags, flag expiry, `shared/` growth, and deprecation deadlines. The measure of success after year one: **a new team ships their first domain feature in under a week without consulting an architect**, CI still finishes affected gates in minutes, and the checklist above reads as ordinary habit rather than ceremony.