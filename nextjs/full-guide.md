# Full-Stack Next.js Architecture Guide (Scalable Enterprise Version)

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions. Nothing here is prescriptive for a particular industry or application.

---

This architecture is designed for:

- Large SaaS systems
- Multi-team development
- Full-Stack Next.js applications (App Router)
- Redux Toolkit + RTK Query (client-side state & server cache)
- Prisma ORM + PostgreSQL + Redis + BullMQ
- Long-term maintainability
- Feature isolation on the frontend
- Domain-driven design on the server side
- Separation of client and server concerns within a single monorepo

The goal is not just folder organization. The real goals are **clear ownership**, **scalability**, **strict boundaries**, **low coupling**, **easier refactoring**, **safe deletion**, **team scalability**, **future microfrontend and microservice readiness**, and **localized UI and domain ownership per module**.

---

# Full-Stack Project Structure

```txt
.env
.env.example
next.config.ts
prisma/
└── schema.prisma                  # Central Prisma ORM schema
src/
│
│  ╔══════════════════════════════════════════════════════════════╗
│  ║         NEXT.JS APP ROUTER  (pages + API entry points)      ║
│  ╚══════════════════════════════════════════════════════════════╝
│
├── app/
│   ├── api/                       # API Route Handlers — thin entry points ONLY
│   │   └── v1/
│   │       ├── items/
│   │       │   └── route.ts       # Delegates to server/modules/…/api/handler.ts
│   │       ├── items/
│   │       │   └── [id]/
│   │       │       └── route.ts
│   │       └── ...
│   ├── (dashboard)/               # Page route groups
│   ├── (auth)/
│   ├── (public)/
│   └── layout.tsx
│
│  ╔══════════════════════════════════════════════════════════════╗
│  ║    SERVER-SIDE  (DDD — Domain-Driven Design Backend)        ║
│  ╚══════════════════════════════════════════════════════════════╝
│
├── server/
│   ├── modules/                   # Domain modules (equivalent to Django's apps/)
│   │   │
│   │   │  # ─── Domain Group: Identity & Access ─────────────────────────────
│   │   ├── (identity)/
│   │   │   ├── users/             # Module: users & auth
│   │   │   └── authorization/     # Module: RBAC (roles, memberships, permissions)
│   │   │
│   │   │  # ─── Domain Group: Core Domain ──────────────────────────────────
│   │   ├── (core-domain)/
│   │   │   ├── organizations/     # Module: tenant/organization management
│   │   │   ├── items/             # Module: core business resource
│   │   │   └── customers/         # Module: customer records
│   │   │
│   │   │  # ─── Domain Group: Operations ─────────────────────────────────────
│   │   ├── (operations)/
│   │   │   ├── orders/            # Module: order processing
│   │   │   ├── scheduling/        # Module: time slots & availability
│   │   │   └── catalog/           # Module: product/service catalog
│   │   │
│   │   │  # ─── Domain Group: Discovery ──────────────────────────────────────
│   │   ├── (discovery)/
│   │   │   └── search/            # Module: full-text search (read-only)
│   │   │
│   │   │  # ─── Domain Group: Integrations ───────────────────────────────────
│   │   ├── (integrations)/
│   │   │   └── integrations/      # Module: third-party adapters
│   │   │
│   │   └── <module>/              # Template — every module follows this layout
│   │       ├── api/
│   │       │   ├── handler.ts     # HTTP controller — thin delegation — zero domain logic
│   │       │   └── schemas.ts     # Zod in/out shapes only
│   │       ├── domain/
│   │       │   ├── entities.ts    # TypeScript domain objects (no ORM)
│   │       │   ├── value-objects.ts # Immutable value types
│   │       │   ├── exceptions.ts  # Domain-specific exceptions
│   │       │   └── rules.ts       # Pure business rule functions / strategies
│   │       ├── services/          # Write use-cases (POST/PUT/PATCH/DELETE)
│   │       │   ├── index.ts       # Re-exports all service classes
│   │       │   └── <entity>-services/  # One folder per domain entity
│   │       │       ├── create-<entity>.service.ts
│   │       │       ├── update-<entity>.service.ts
│   │       │       └── delete-<entity>.service.ts
│   │       ├── repositories/
│   │       │   └── prisma.repo.ts # Single concrete Prisma ORM implementation
│   │       ├── cache/             # Only for modules that need real-time Redis state
│   │       │   └── redis.repo.ts
│   │       ├── tasks/             # Thin BullMQ job wrappers (one file per task)
│   │       └── tests/
│   │           ├── unit/          # Domain + service unit tests (no DB)
│   │           └── integration/   # Route handler + DB integration tests
│   │
│   ├── core/
│   │   ├── exceptions.ts          # Base ApplicationError + HTTP status mapping
│   │   ├── responses.ts           # Standardized API response helpers
│   │   ├── permissions.ts         # RBAC resolver and requiresPermission wrapper
│   │   ├── handler-wrapper.ts     # withErrorHandler — global error boundary
│   │   └── pagination.ts          # Shared pagination helpers
│   │
│   └── config/
│       ├── db.ts                  # Prisma client singleton
│       ├── redis.ts               # Redis (ioredis) client singleton
│       └── env.ts                 # Environment validation (Zod)
│
│  ╔══════════════════════════════════════════════════════════════╗
│  ║  FRONTEND  (React / Next.js Client-Side Architecture)       ║
│  ╚══════════════════════════════════════════════════════════════╝
│
├── bootstrap/
│
├── core/
│
├── api/
│   ├── base/
│   ├── generated/
│   ├── contracts/
│   ├── mappers/
│   └── index.ts
│
├── layouts/
│   ├── (group-1)/
│   │   ├── dashboard-layout/
│   │   └── ...
│   ├── (group-2)/
│   │   └── auth-layout/
│   ├── (group-3)/
│   │   └── public-layout/
│   └── index.ts
│
├── shared/
│   ├── ui/
│   ├── hooks/
│   ├── utils/
│   ├── lib/
│   ├── constants/
│   ├── types/
│   ├── icons/
│   ├── assets/
│   └── i18n/
│
├── modules/
│   ├── (group-1)/
│   │   ├── module-1/
│   │   └── ...
│   ├── (group-2)/
│   └── (group-3)/
│
├── routes/
│
├── store/
│
└── styles/
```

> **Environment Files**
>
> `.env` holds actual configuration values. `.env.example` must be kept in sync whenever keys are added or removed, but only placeholder (non-sensitive) values should be committed.

---

---

# PART 1 — FRONTEND ARCHITECTURE

> This section is the complete React / Next.js frontend architecture guide. All content below in Part 1 applies to the client-side layer of the application.

---

# Frontend Architecture (Scalable Enterprise Version)

This architecture is designed for:

* Large SaaS systems
* Multi-team development
* React applications across the ecosystem
* React / Next.js applications
* Redux Toolkit + RTK Query
* Long-term maintainability
* Feature isolation
* Domain-driven frontend systems

The goal is not just folder organization. The real goals are **clear ownership**, **scalability**, **strict boundaries**, **low coupling**, **easier refactoring**, **safe deletion**, **team scalability**, **future microfrontend readiness**, and **localized UI ownership per module and feature**.

This architecture is also not limited to routes. The same grouping concept can be used to organize **features, pages, layouts, modules, and other related units** wherever it improves clarity and ownership.

---

# Frontend Project Structure

```txt
.env
.env.example
src/
│
├── app/ # routes(pages) in case  using nextjs
│
├── bootstrap/
│
├── core/
│
├── api/
│   ├── base/
│   ├── generated/
│   ├── contracts/
│   ├── mappers/
│   └── index.ts
│
├── layouts/
│   ├── (group-1)/
│   │   ├── dashboard-layout/
│   │   │   ├── i18n/
│   │   │   ├── components/
│   │   │   ├── sections/
│   │   │   ├── application/
│   │   │   ├── state/
│   │   │   ├── hooks/
│   │   │   ├── utils/
│   │   │   ├── validation/
│   │   │   ├── types/
│   │   │   ├── constants/
│   │   │   ├── tests/
│   │   │   ├── domain/
│   │   │   └── index.ts
│   │   └── ...
│   │
│   ├── (group-3)/
│   │   └── auth-layout/
│   │
│   ├── (group-2)/
│   │   └── public-layout/
│   │
│   └── index.ts
│
├── shared/
│   ├── ui/
│   ├── hooks/
│   ├── utils/
│   ├── lib/
│   ├── constants/
│   ├── types/
│   ├── icons/
│   ├── assets/
│   └── i18n/ # Global config and utils of i18n
│
├── modules/
│   ├── (group-1)/
│   │   ├── module-1/
│   │   │   ├── i18n/
│   │   │   ├── pages/
│   │   │   ├── domain/
│   │   │   ├── shared/
│   │   │   │
│   │   │   ├── feature-1/
│   │   │   │   ├── domain/ # Feature-bound domain logic
│   │   │   │   ├── i18n/
│   │   │   │   ├── components/
│   │   │   │   ├── sections/
│   │   │   │   ├── application/
│   │   │   │   ├── state/
│   │   │   │   ├── hooks/
│   │   │   │   ├── utils/
│   │   │   │   ├── validation/
│   │   │   │   ├── types/
│   │   │   │   ├── constants/
│   │   │   │   ├── tests/
│   │   │   │   └── index.ts
│   │   │   │
│   │   │   ├── feature-2/
│   │   │   └── ...
│   │   │   │
│   │   │   └── index.ts
│   │   │
│   │   ├── module-2/
│   │   │   └── ...
│   │   └── module-3/
│   │
│   ├── (group-2)/
│   │   └── ...
│   │
│   └── (group-3)/
│       └── ...
│
├── routes/
│
├── store/
│
├── styles/
│
└── main.tsx
```

> **Environment Files**
>
> `.env` holds the actual configuration values. `.env.example` must be kept in sync with `.env` whenever keys are added or updated, but only example (non-sensitive) values should be committed.

---

# Group Folders

Group folders are folders wrapped in parentheses, for example:

* `(dashboard)`
* `(public)`
* `(auth)`

They are used to **group related code without affecting the URL or public structure** when the framework or architecture supports that convention. In this architecture, the idea is broader than routing: it can also be used to group **pages, features, layouts, modules, and other related business areas**.

### Why use group folders

* Keep related code together without leaking it into the public path structure
* Separate dashboard, public, and auth concerns cleanly
* Group pages, features, layouts, and modules by business area
* Make ownership clearer for teams
* Improve maintainability as the app grows
* Avoid mixing unrelated responsibilities in the same tree
* Support different layout shells without duplication
* Reduce cognitive load by making bounded contexts visible in the folder name

### Example intent

```txt
src/
├── modules/
│   ├── (dashboard)/
│   │   ├── orders/
│   │   ├── procducts/
│   │   └── team/
│   └── (public)/
│       ├── landing/
│       └── pricing/
│
└── layouts/
    ├── (dashboard)/
    ├── (public)/
    └── (auth)/
```

The parentheses indicate that the folder is a **group folder**, not a business endpoint and not necessarily part of the final runtime path.

---

# Frontend Layer Responsibilities

## Root & Infrastructure Layers

| Folder / Layer | Description & Responsibilities                                                       | Allowed / Contains                                                                                           | Forbidden                                                          |
| -------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| app/           | Application composition layer. Handles global setup and entry wrappers.              | Providers, App composition, Initialization, Theme setup, Router integration, Error boundaries.               | Business logic, Feature workflows, Domain rules, Feature state.    |
| bootstrap/     | Application startup logic executed before rendering the app.                         | Environment setup, Feature flags initialization, Startup configuration, Service initialization, Polyfills.   | UI components, Application layouts, React hooks.                   |
| core/          | Infrastructure layer providing the system's technical, business-agnostic foundation. | HTTP clients, Auth infrastructure, Permissions engine, Logging, Monitoring, Analytics, Storage abstractions. | UI presentation code, Business-specific models, Feature execution. |
| api/base/      | Low-level network/API infrastructure.                                                | Axios instances, Fetch wrappers, Interceptors, Retry strategies, Refresh token logic, Error normalization.   | Business rules, Feature-specific logic.                            |
| api/generated/ | Auto-generated backend definitions. **Must remain immutable.**                       | Swagger/OpenAPI generated types, Generated API clients, Raw DTOs.                                            | Manual modifications, Custom business logic mapping.               |
| api/contracts/ | Frontend-facing API abstractions protecting the UI from breaking DTO modifications.  | Frontend API contracts, Normalized response models, Request abstractions, Contract wrappers.                 | UI-specific logic, React hooks.                                    |
| api/mappers/   | Transformation layer converting structures between client and server worlds.         | mapUserDtoToUser(), mapAppointmentResponse(), DTO ↔ Domain Model converters.                                 | React primitives, Global state interactions.                       |
| layouts/       | Application shell layer mapping core layout concerns.                                | Dashboard layouts, Auth layouts, Public layouts, Navigation shells, Page containers.                         | Business workflows, API calls, Thick state management.             |

---

## Global & Domain Shared Layers

| Folder / Layer | Description & Responsibilities                                                     | Allowed / Contains                                                                                                 | Forbidden                                                                  |
| -------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| shared/        | Purely technical global reusable items completely stripped of business awareness.  | Generic components (shared/ui/Button), useDebounce, pure utilities (date, currency), global configuration (i18n/). | Domain-aware items (useDoctorData, UserProfileCard), business constraints. |
| modules/       | Business domain layer. Each subfolder represents a fully isolated bounded context. | Independent folders for domain topics (e.g., billing, appointments, notifications).                                | Cross-module direct file dependencies without going through public APIs.   |
| module/i18n/   | Module-level localization files shared across multiple feature modules.            | Shared page titles, Shared labels, Notification templates, Module-wide validation strings.                         | Feature-exclusive strings, Domain-agnostic generic strings.                |
| module/pages/  | High-level page layout configurations and feature composition.                     | Layout compositions, route composition, page orchestration.                                                        | Directly handling API calls, complicated data orchestration.               |
| module/domain/ | Pure, framework-independent business core logic.                                   | Entities, Core business rules, Domain services, Pure transformations (calculateInvoice).                           | React components, Redux logic, API requests, Browser API dependencies.     |
| module/shared/ | Reusable utilities constrained strictly within this specific business module.      | Shared UI components, Hooks, and Constants unique to this module's features.                                       | Infrastructure wrappers, Globally generic components.                      |

---

## Feature-Level Layers (modules/*-module/feature-x/*)

| Folder / Layer       | Description & Responsibilities                                                               | Allowed / Contains                                                                      | Forbidden                                                                          |
| -------------------- | -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| feature/domain/      | Feature-local domain and business rules required strictly to satisfy this isolated use case. | Feature specific rules, validations, and domain-scoped calculations.                    | React context, Hook hooks, Global actions.                                         |
| feature/i18n/        | Micro-localization files exclusive to this particular use case.                              | Feature-specific button titles, Alert messages, Inline helper micro-copy.               | Shared global/module strings.                                                      |
| feature/components/  | Granular UI building blocks isolated inside the local feature space.                         | Small presentational structures, Form inputs, Simple layout parts.                      | Network logic, Global state synchronization.                                       |
| feature/sections/    | Large UI structures orchestrating presentational elements.                                   | FormSection, TableSection, Layout assembly components.                                  | Complex domain logic, Root framework integrations.                                 |
| feature/application/ | Workflow coordinator and business orchestration engine.                                      | Business workflows, Form submit orchestration, Modal toggles, Optimistic state updates. | Low-level HTML structure / Primitive rendering.                                    |
| feature/state/       | Feature-specific client-side data state managers.                                            | Redux slices, Zustand stores, Selectors, Local derived metrics.                         | Duplicating server state cache from RTK Query.                                     |
| feature/hooks/       | Hooks scoping behaviors down to this single unique application scope.                        | useFeaturePermissions, useFeatureFilters, useFeatureActions.                            | Globally generic utility functions.                                                |
| feature/utils/       | Functional stateless helpers serving local components.                                       | Deterministic, side-effect-free data manipulation helpers.                              | Managing state configurations, Browser mutations.                                  |
| feature/validation/  | Contract constraints evaluating runtime accuracy before processing actions.                  | Zod configurations, Yup schemas, Form checking workflows.                               | Shared cross-domain schemas.                                                       |
| feature/types/       | Data definitions defining shapes internal to this layout piece.                              | Component props types, Local state payload structures.                                  | Sharing out of feature boundaries without elevating definition up to module scope. |
| feature/constants/   | Hardcoded parameters defining behavioral elements inside this component.                     | Action configurations, Static lists, Grid layout configurations.                        | Global system parameters.                                                          |
| feature/tests/       | Continuous verification suite ensuring this unit runs correctly.                             | Unit tests, Component integration workflows, Performance tracking files.                | Cross-module logic testing configurations.                                         |
| feature/index.ts     | Public API boundary exposing designated functional capabilities outward.                     | Explicit item exports defining what other local folders can consume.                    | Exposing deep implementation details (export * from './everything').               |

---

# Frontend Architectural Rules

* **Rule 1 (No Reverse Imports):** Cross-feature direct imports are strictly forbidden. feature-1 must never import directly from feature-2.
* **Rule 2 (Feature Isolation):** Features should remain fully autonomous. Share code between features only by bubbling it up to module/shared/, the module/domain/ layer, or orchestration patterns.
* **Rule 3 (Strict Downward Flow):** shared/ must remain completely business-agnostic and never import from modules/ or core/.
* **Rule 4 (Domain Purity):** The domain/ layer must stay 100% pure: no React components, no hooks, no state managers, and no direct API calls.
* **Rule 5 (Thin View Layers):** Pages and components must keep business logic out of their views. JSX files should focus strictly on presentational mapping.
* **Rule 6 (Pragmatic State Management):** Choose state placement deliberately based on data lifecycle rules:

| State Type                | Recommended Tool      |
| ------------------------- | --------------------- |
| **Local UI state**        | useState / useReducer |
| **Server cache**          | RTK Query             |
| **Global app state**      | Redux Toolkit         |
| **Complex feature state** | Zustand / Redux       |

* **Rule 7 (Server State Source of Truth):** RTK Query manages server caching. Never manually sync or duplicate raw server responses into global Redux slices.
* **Rule 8 (Controlled Public Surfaces):** Avoid blind barrel exports (export * from './everything'). Explicitly list public APIs to avoid circular dependencies and ensure clean tree-shaking.
* **Rule 9 (Safe Deletability):** If dropping a feature directory breaks an unrelated area of the app, your domain boundaries are bleeding.
* **Rule 10 (Encapsulate Declarative Checks):** Do not write loose boolean conditions inside UI elements (e.g., user.role === 'ADMIN' && clinic.status === 'ACTIVE'). Wrap conditions in descriptive helper functions like canManageClinic(user, clinic).
* **Rule 11 (Presentation-First UI):** UI components should remain dumb and predictable. They accept parameters via props, emit events through callbacks, and delegate logic upward.
* **Rule 12 (Module Gateways):** Every domain module must maintain an explicit gateway file via modules/[domain-module]/index.ts. External layers must always go through this public contract.
* **Rule 13 (No Architectural Dumping Grounds):** Folder names like misc/, helpers/, common/, or other/ are strictly banned.
* **Rule 14 (DTO Sandbox Boundary):** Backend API contracts must never leak straight into UI elements. Always route inputs through mappers and local frontend contracts first.
* **Rule 15 (Lightweight Layout Frameworks):** Layout structures function purely as application layout wrappers, never as processing yards for data operations.
* **Rule 16 (Business-Blind Infrastructure):** Core configuration frameworks like core/ or api/base/ must never contain any code that understands your specific business domain.
* **Rule 17 (Distributed Localization):** Keep copy files adjacent to the logic that uses them:

  * feature/i18n/ handles strings unique to that exact use case.
  * module/i18n/ manages copy shared across a domain context.
  * shared/i18n/ handles truly generic UI words (e.g., "Save", "Cancel").

---

# Frontend Final Architectural Performance Metrics

| Area                        | Result    | Target Metric Met                                                          |
| --------------------------- | --------- | -------------------------------------------------------------------------- |
| **Scalability**             | High      | Unlocked parallel workflows for multi-team execution.                      |
| **Maintainability**         | High      | Code changes stay contained inside localized directories.                  |
| **Team Collaboration**      | Excellent | Minimized codebase conflicts via clear feature boundaries.                 |
| **Domain Isolation**        | Strong    | Core business rules are protected from UI framework churn.                 |
| **Refactoring Safety**      | High      | Isolated scopes make code deletion safe and reliable.                      |
| **Feature Ownership**       | Clear     | Modules track perfectly to business domains and engineering teams.         |
| **Testing**                 | Easier    | Pure business layers accept straightforward unit testing.                  |
| **Cognitive Load**          | Lower     | Developers only need to reason about a single feature directory at a time. |
| **Microfrontend Readiness** | Excellent | Features are prepared for future decomposition into standalone micro-apps. |
| **Localization Ownership**  | Clear     | Translation updates stay coupled with their respective features.           |

---

---

# PART 2 — SERVER-SIDE ARCHITECTURE

> This section governs the `src/server/` directory and the thin API entry points in `src/app/api/`. All patterns mirror the Django guide's domain structure, adapted fully to TypeScript, Prisma, Zod, and Next.js Route Handlers.

---

# Server-Side Project Structure

```txt
src/server/
│
├── modules/                        # Domain modules (equivalent to Django's apps/)
│   │
│   │   # ─── Domain Group: Identity & Access ──────────────────────────────
│   ├── (identity)/
│   │   ├── users/                  # Module: users & auth
│   │   └── authorization/          # Module: RBAC
│   │
│   │   # ─── Domain Group: Core Domain ────────────────────────────────────
│   ├── (core-domain)/
│   │   ├── organizations/
│   │   ├── items/
│   │   └── customers/
│   │
│   │   # ─── Domain Group: Operations ─────────────────────────────────────
│   ├── (operations)/
│   │   ├── orders/
│   │   ├── scheduling/
│   │   └── catalog/
│   │
│   │   # ─── Domain Group: Discovery ──────────────────────────────────────
│   ├── (discovery)/
│   │   └── search/
│   │
│   │   # ─── Domain Group: Integrations ───────────────────────────────────
│   ├── (integrations)/
│   │   └── integrations/
│   │
│   └── <module>/                   # Template — every module follows this layout
│       ├── api/
│       │   ├── handler.ts          # HTTP controller — thin delegation only
│       │   └── schemas.ts          # Zod in/out shapes only
│       ├── domain/
│       │   ├── entities.ts         # TypeScript domain interfaces (no ORM)
│       │   ├── value-objects.ts    # Immutable value types
│       │   ├── exceptions.ts       # Domain-specific exceptions
│       │   └── rules.ts            # Pure business rule functions / strategies
│       ├── services/               # Write use-cases (POST/PUT/PATCH/DELETE)
│       │   ├── index.ts            # Re-exports all service classes
│       │   └── <entity>-services/  # One folder per domain entity
│       │       ├── create-<entity>.service.ts
│       │       ├── update-<entity>.service.ts
│       │       └── delete-<entity>.service.ts
│       ├── repositories/
│       │   └── prisma.repo.ts      # Single concrete Prisma ORM implementation
│       ├── cache/                  # Only for modules needing real-time Redis state
│       │   └── redis.repo.ts
│       ├── tasks/                  # Thin BullMQ job wrappers (one file per task)
│       └── tests/
│           ├── unit/               # Domain + service unit tests (no DB)
│           └── integration/        # Route handler + DB integration tests
│
├── core/
│   ├── exceptions.ts               # Base ApplicationError + HTTP status mapping
│   ├── responses.ts                # Standardized API response helpers
│   ├── permissions.ts              # RBAC resolver and requiresPermission wrapper
│   ├── handler-wrapper.ts          # withErrorHandler — global error boundary
│   └── pagination.ts               # Shared pagination helpers
│
└── config/
    ├── db.ts                       # Prisma client singleton
    ├── redis.ts                    # Redis (ioredis) client singleton
    └── env.ts                      # Environment validation (Zod)
```

```txt
src/app/api/v1/                     # Next.js API Route Handlers — thin entry points
├── items/
│   └── route.ts                    # export { GET, POST } ← from server/modules/items/api/handler.ts
├── items/[id]/
│   └── route.ts                    # export { GET, PUT, DELETE }
└── ...
```

> **The `src/app/api/` directory** is the Next.js HTTP entry layer. It is the equivalent of Django's `urls.py`. Every `route.ts` file must only re-export named handlers from `server/modules/<module>/api/handler.ts`. Zero logic lives here.

---

# Domain Groups

Domain groups are **conceptual and structural boundaries** inside `server/modules/` that cluster related modules by business area. In Next.js/TypeScript, they are expressed as subdirectories using the parenthetical naming convention — identical to Next.js route groups, but applied to the server layer.

**Why use domain groups:**

- Keep related bounded contexts visible in the codebase
- Separate identity, core business, operations, and discovery concerns cleanly
- Make ownership clearer for teams
- Improve maintainability as the project grows
- Reduce cognitive load by making bounded contexts explicit

**Standard domain groups:**

| Domain Group       | Example Modules                              | Responsibilities                                          |
| ------------------ | -------------------------------------------- | --------------------------------------------------------- |
| Identity & Access  | users, authorization                         | Auth, users, RBAC, roles, memberships                     |
| Core Domain        | organizations, items, customers              | Tenant management, core resources, customer records       |
| Operations         | orders, scheduling, catalog                  | Order processing, time slots, product/service catalog     |
| Discovery          | search                                       | Full-text search, public browsing                         |
| Integrations       | integrations/*                               | Third-party system adapters                               |

When adding a new module, assign it to an existing domain group, or establish a new group if it represents a genuinely independent bounded context with its own entities, writes, and reads.

---

# Naming Conventions

All naming across the entire server-side codebase must be in **English**.

---

## File Naming Rules

- All file names use **`kebab-case`**.
- Service files follow the pattern: **`<verb>-<entity>.service.ts`**
  - `create-order.service.ts`, `update-user.service.ts`, `delete-item.service.ts`
- Repository files: **`prisma.repo.ts`**, **`redis.repo.ts`** (one per module, named by infrastructure)
- API files: **`handler.ts`**, **`schemas.ts`** (always these exact names inside `api/`)
- Domain files: **`entities.ts`**, **`value-objects.ts`**, **`exceptions.ts`**, **`rules.ts`**
- Task files: **`<descriptive-action>.task.ts`**
  - `send-welcome-email.task.ts`, `process-order.task.ts`
- Test files: **`<feature>.<layer>.test.ts`**
  - `order-service.unit.test.ts`, `item-handler.integration.test.ts`

---

## Class Naming Rules

- All class names use **`PascalCase`**.
- Prisma-generated model types: **`<Entity>`** (auto-generated by Prisma from schema)
  - `User`, `Order`, `Organization`, `Item` (Prisma model types from `@prisma/client`)
- Domain entity interfaces: **`<Entity>`** (matching Prisma names — keep them identical)
  - `User`, `Order`, `Organization`, `Item`
- Repository classes: **`Prisma<Entity>Repository`** or **`Redis<Entity>Repository`**
  - `PrismaUserRepository`, `PrismaOrderRepository`, `RedisAuthorizationRepository`
- Service classes: **`<Verb><Entity>Service`**
  - `CreateOrderService`, `UpdateUserService`, `DeleteItemService`, `LoginService`
- Zod schema names: **`create<Entity>Schema`**, **`<entity>ResponseSchema`**, **`update<Entity>Schema`**
  - `createOrderSchema`, `orderResponseSchema`, `updateItemSchema`
- Exception classes: **`<Entity><Condition>Error`**
  - `OrderNotFoundError`, `UserAlreadyExistsError`, `SlotUnavailableError`
- Value Object classes: **`<Concept>`**
  - `Email`, `PhoneNumber`, `Money`, `Slug`
- Strategy abstract classes: **`<Concept>Strategy`**
  - `PriorityStrategy`, `PricingStrategy`, `DiscountStrategy`

---

## Method Naming Rules

- All method names use **`camelCase`**.
- Repository read methods: **`getBy<Field>`**, **`listBy<Field>`**, **`existsBy<Field>`**
  - `getById()`, `listByOrganization()`, `existsByEmail()`
- Repository write methods: **`save`**, **`update`**, **`delete`**, **`bulkCreate`**
- Service public method: **`execute`** — always this exact name; the class name carries the intent
- Domain rule functions: **`can<Action>`**, **`calculate<Result>`**, **`validate<Condition>`**
  - `canCancelOrder()`, `calculatePriority()`, `validateMembership()`
- Mapper methods (private): **`_mapToDomain`**, **`_mapToORM`**

---

## Variable Naming Rules

- All variable names use **`camelCase`**.
- Boolean variables: **`is<Condition>`**, **`has<Feature>`**, **`can<Action>`**
  - `isActive`, `hasPermission`, `canCancel`
- Collections: plural form of the entity
  - `orders`, `users`, `items`
- IDs: **`<entity>Id`**
  - `orderId`, `userId`, `organizationId`
- Timestamps: **`<event>At`** or **`<event>On`**
  - `createdAt`, `updatedAt`, `expiresOn`

---

## URL / Endpoint Naming Rules

- URL segments use **`kebab-case`**.
- Plural nouns for collections: **`/api/v1/orders`**
- Nested resources: **`/api/v1/organizations/[orgId]/members`**
- Specific actions on a resource: **`/api/v1/orders/[orderId]/cancel`**

---

## Redis Key Naming Rules

- All Redis key segments use **English**.
- Pattern: **`<domain>:<id>:<type>`**
- Examples:
  - `queue:{queueId}:counter` → STRING — atomic INCR
  - `queue:{queueId}:state` → HASH — live state
  - `slots:{orgId}:{catalogId}:{date}` → HASH — TTL 60s
  - `perm:{orgId}:{userId}` → STRING — TTL 300s
  - `search:{orgId}:{query}:{month}` → STRING — TTL 3600s
  - `<domain>:{id}:state` → HASH — live state snapshot
  - `pubsub:<domain>:{id}` → CHANNEL — entity-scoped events
  - `session:<domain>:<code>` → STRING — short-lived (with TTL)

---

## Permission String Naming Rules

- Pattern: **`<resource>:<action>`**
- All lowercase, colon-separated.
- No wildcards. No `manage` shorthand — always expand to discrete verbs.
- Examples:
  - `orders:create`, `orders:read`, `orders:update`, `orders:delete`
  - `orders:own:read` — row-level scoping
  - `organization:settings:write`, `organization:settings:read`
  - `members:invite`, `members:deactivate`, `members:read`
  - `roles:create`, `roles:update`, `roles:delete`

---

## Schema Field Naming Rules

- All Zod schema field names use **`camelCase`**.
- Follow the same rules as variable naming above.
- Examples: `orderId`, `status`, `createdAt`, `isActive`, `durationMin`

---

## Prisma Model Field Naming Rules

- All Prisma model field names use **`camelCase`** (Prisma convention).
- Foreign key fields: **`<entity>Id`** (e.g., `organizationId`, `userId`)
- Examples: `ticketNumber`, `status`, `createdAt`, `isActive`, `durationMin`

---

# Standardized API Response Structure

All API endpoints must return a unified, predictable JSON structure for both successful requests and errors. This ensures frontend clients, mobile apps, and third-party integrations can parse responses consistently.

---

## Core Response Types

The standard response types and factory helpers are defined in `server/core/responses.ts`.

```typescript
// Example only — adapt to your project

// src/server/core/responses.ts

export interface ErrorDetail {
  field?: string;
  message: string;
  code?: string; // Machine-readable error code (e.g., "required", "invalid_format")
}

export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T | null;
}

export interface PaginatedData<T> {
  items: T[];
  total: number;
  page: number;
  size: number;
  pages: number;
}

export interface ErrorResponse {
  success: false;
  message: string;
  errors?: ErrorDetail[];
}

// ── Factory Helpers ───────────────────────────────────────────────────────────

export function createSuccessResponse<T>(
  data: T,
  message = 'Request successful',
): ApiResponse<T> {
  return { success: true, message, data };
}

export function createPaginatedResponse<T>(
  items: T[],
  total: number,
  page: number,
  size: number,
  message = 'Items retrieved successfully',
): ApiResponse<PaginatedData<T>> {
  return {
    success: true,
    message,
    data: { items, total, page, size, pages: Math.ceil(total / size) },
  };
}

export function createErrorResponse(
  message: string,
  errors?: ErrorDetail[],
): ErrorResponse {
  return { success: false, message, errors };
}
```

---

## Success Response Contract

**Single Item Response:**
```json
{
  "success": true,
  "message": "Item retrieved successfully",
  "data": {
    "id": "uuid",
    "name": "Example Item",
    "isActive": true
  }
}
```

**List Response (Paginated):**
```json
{
  "success": true,
  "message": "Items retrieved successfully",
  "data": {
    "items": [{ "id": "uuid", "name": "Item 1", "isActive": true }],
    "total": 50,
    "page": 1,
    "size": 20,
    "pages": 3
  }
}
```

**Deletion / Empty Data Response:**
```json
{
  "success": true,
  "message": "Item deleted successfully",
  "data": null
}
```

---

## Error Response Contract

**Business Logic Error (400, 403, 404, 409):**
```json
{
  "success": false,
  "message": "Order cannot be cancelled",
  "errors": [
    {
      "field": null,
      "message": "Orders that are already shipped cannot be cancelled",
      "code": "order_already_shipped"
    }
  ]
}
```

**Validation Error (422):**
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "Invalid email format", "code": "invalid_string" },
    { "field": "quantity", "message": "Number must be greater than 0", "code": "too_small" }
  ]
}
```

---

## Global Error Handler (withErrorHandler)

Unlike Django-Ninja's global exception handler, Next.js Route Handlers require an explicit wrapper. Every handler function must be wrapped with `withErrorHandler`.

```typescript
// Example only — adapt to your project

// src/server/core/handler-wrapper.ts

import { NextRequest, NextResponse } from 'next/server';
import { ZodError } from 'zod';
import { ApplicationError } from './exceptions';
import { createErrorResponse, ErrorDetail } from './responses';

type RouteHandler<C = unknown> = (
  req: NextRequest,
  ctx?: C,
) => Promise<NextResponse>;

export function withErrorHandler<C = unknown>(
  handler: RouteHandler<C>,
): RouteHandler<C> {
  return async (req: NextRequest, ctx?: C) => {
    try {
      return await handler(req, ctx);
    } catch (error) {
      if (error instanceof ApplicationError) {
        return NextResponse.json(
          createErrorResponse(error.message, [
            { field: error.field, message: error.message, code: error.code },
          ]),
          { status: error.statusCode },
        );
      }

      if (error instanceof ZodError) {
        const errors: ErrorDetail[] = error.errors.map((e) => ({
          field: e.path.join('.'),
          message: e.message,
          code: e.code,
        }));
        return NextResponse.json(
          createErrorResponse('Validation failed', errors),
          { status: 422 },
        );
      }

      console.error('[UnhandledError]', error);
      return NextResponse.json(
        createErrorResponse('Internal server error'),
        { status: 500 },
      );
    }
  };
}
```

---

## Base Domain Exception

```typescript
// Example only — adapt to your project

// src/server/core/exceptions.ts

export class ApplicationError extends Error {
  constructor(
    public readonly message: string = 'An application error occurred',
    public readonly statusCode: number = 400,
    public readonly code?: string,
    public readonly field?: string,
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NotFoundError extends ApplicationError {
  constructor(message = 'Resource not found', field?: string) {
    super(message, 404, 'not_found', field);
  }
}

export class ConflictError extends ApplicationError {
  constructor(message = 'Resource conflict', field?: string) {
    super(message, 409, 'conflict', field);
  }
}

export class ForbiddenError extends ApplicationError {
  constructor(message = 'Permission denied', field?: string) {
    super(message, 403, 'forbidden', field);
  }
}
```

---

# Server-Side Layer Responsibilities

## Infrastructure & Configuration Layers

| Layer                     | Description & Responsibilities                                              | Allowed / Contains                                                                                 | Forbidden                                                         |
| ------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| server/config/db.ts       | Prisma client singleton.                                                    | `new PrismaClient()`, singleton export.                                                            | Business logic, queries, domain rules.                            |
| server/config/redis.ts    | Redis (ioredis) client singleton.                                           | `new Redis(process.env.REDIS_URL)`, singleton export.                                              | Business logic, cache strategies.                                 |
| server/config/env.ts      | Environment variable validation and typed access.                           | Zod schema validating all env vars, typed `env` export.                                            | Application logic, module imports.                                |
| server/core/              | Infrastructure layer providing the system's technical, agnostic foundation. | Base exception hierarchy, response wrappers, RBAC resolver, permission wrapper, pagination helpers. | Business-specific models, feature execution, domain rules.        |
| server/core/exceptions.ts | Base error hierarchy that all domain exceptions extend.                     | `ApplicationError`, `NotFoundError`, `ConflictError`, `ForbiddenError`.                            | Feature-specific exception subclasses (those live in each module's `domain/exceptions.ts`). |
| server/core/responses.ts  | Standardized API response types and factory helpers.                        | `ApiResponse<T>`, `PaginatedData<T>`, `ErrorResponse`, factory functions.                          | Business logic, domain imports.                                   |
| server/core/permissions.ts| Single source of truth for all RBAC resolution.                             | `requiresPermission()` wrapper, `hasPermission()` resolver, live DB permission lookup.             | Business logic beyond permission checking.                        |
| server/core/handler-wrapper.ts | Global error boundary for all route handlers.                          | `withErrorHandler()` HOF, error-to-response mapping.                                               | Business logic, domain imports.                                   |
| app/api/v1/**/route.ts   | Next.js HTTP entry points. Thin re-export layer only.                       | Named HTTP verb exports (`GET`, `POST`, `PUT`, `DELETE`) delegating to handler.ts.                 | Any logic whatsoever. These files are re-exports only.            |

---

## Module-Level Layers

| Layer                       | Description & Responsibilities                                                           | Allowed / Contains                                                                                                           | Forbidden                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| api/handler.ts              | HTTP controller. Reads → call repository directly. Writes → call service.                | Wrapped handler functions, Zod `.parse()` calls, `requiresPermission()`, `createSuccessResponse()`, direct repo/service imports. | Business logic, domain rules, Prisma model imports, transaction management.             |
| api/schemas.ts              | Zod input/output shapes for the HTTP boundary.                                           | Request schemas (`z.object()`), response type aliases, `z.infer<>` type exports.                                             | Domain entities, Prisma models, business constraint validation.                         |
| domain/entities.ts          | Pure, framework-independent domain interfaces.                                           | TypeScript interfaces and types, computed properties, domain-level constraints.                                              | Prisma imports, Redis, BullMQ, any I/O.                                                 |
| domain/value-objects.ts     | Immutable, self-validating types wrapping primitives.                                    | `Email`, `PhoneNumber`, `Slug` and similar types with built-in validators.                                                   | Mutable state, Prisma references.                                                       |
| domain/exceptions.ts        | Domain-specific exception classes.                                                       | `OrderNotFoundError`, `SlotUnavailableError` — all subclasses of `core/exceptions.ApplicationError`.                         | HTTP status codes, Next.js imports.                                                     |
| domain/rules.ts             | Pure business rule functions and strategy abstractions.                                  | `canCancelOrder()`, `calculatePriority()`, `PriorityStrategy` abstract class and subclasses.                                  | I/O operations, Prisma access, service calls.                                           |
| services/                   | Write orchestration layer. One class per use-case, one method: `execute()`.              | Domain imports, repository imports, cache imports, `prisma.$transaction()`, post-commit side effects.                        | `api/` imports, HTTP objects, Prisma model imports (`@prisma/client` raw models).        |
| repositories/prisma.repo.ts | Concrete Prisma data-access implementation. Owns all reads and writes for this module.   | Prisma queries, `_mapToDomain()` mappers, `_mapToORM()` converters. Read and write methods clearly sectioned.                | Business logic, validation, HTTP concerns, service calls.                               |
| cache/redis.repo.ts         | Redis interaction layer. Present only in modules that maintain live real-time state.     | Redis read/write operations, key construction, TTL management, atomic operations.                                            | Services, repositories, API layer.                                                      |
| tasks/                      | Thin BullMQ job wrappers. Max 15 lines per task file.                                    | BullMQ `Worker` or `Queue` definitions, service class import and `execute()` call.                                           | Business logic, Prisma queries, multi-step orchestration.                               |
| tests/unit/                 | Fast, isolated unit tests with no DB or network I/O.                                    | Domain entity tests, business rule tests, pure function tests, service tests with mocked repos.                              | Live Prisma, HTTP calls, real Redis.                                                    |
| tests/integration/          | Full-stack tests against the real DB.                                                   | Route handler tests, service-to-DB flow tests.                                                                               | Production Redis, external third-party calls.                                           |

---

# API Documentation Strategy

Next.js does not have built-in OpenAPI generation. The recommended approach is `next-swagger-doc` or `zod-to-openapi`. All route handler files must include JSDoc-style comments that describe the endpoint.

---

## Handler Documentation Rules

Every handler function must have a JSDoc comment describing what it does.

```typescript
// Example only — adapt to your project

// src/server/modules/(core-domain)/items/api/handler.ts

/**
 * GET /api/v1/items
 *
 * List all active items for an organization.
 * Returns a paginated list filtered by organizationId.
 *
 * @query organizationId - UUID of the organization
 * @returns ApiResponse<PaginatedData<ItemResponse>>
 */
export const handleListItems = withErrorHandler(async (req: NextRequest) => {
  // ...
});
```

---

## Response Type Annotations

All handler functions must be typed with explicit return types. Zod schemas must be inferred and re-exported for use by the frontend `api/generated/` layer.

```typescript
// Example only — adapt to your project

// src/server/modules/(core-domain)/items/api/schemas.ts

import { z } from 'zod';

export const createItemSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().optional(),
  organizationId: z.string().uuid(),
});

export const itemResponseSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  description: z.string().optional(),
  isActive: z.boolean(),
  organizationId: z.string().uuid(),
  createdAt: z.string().datetime(),
});

// These types are consumed by both handler.ts and the frontend api/contracts/ layer
export type CreateItemInput = z.infer<typeof createItemSchema>;
export type ItemResponse = z.infer<typeof itemResponseSchema>;
```

---

## API Documentation Rules Summary

1. **Every handler function must have a JSDoc comment** — first line is the summary, subsequent lines are description.
2. **Every handler must declare its Zod schema** for all validated inputs.
3. **Every `POST`/`PUT`/`PATCH` handler must call `schema.parse(body)`** before any other logic.
4. **Response types must be consistent** — always use `createSuccessResponse()`, `createPaginatedResponse()`, or `createErrorResponse()` from `server/core/responses.ts`.
5. **All `route.ts` files must only re-export** from `handler.ts`. Never define logic in `route.ts`.
6. **Zod schema types (`z.infer<>`) must be exported** from `schemas.ts` for consumption by the frontend `api/contracts/` layer.
7. **Mark deprecated handlers** with `@deprecated` JSDoc tag and document the replacement.

---

# SOLID Principles — Enforced Patterns

## Single Responsibility

One service class per use-case, one public method: `execute()`.

```typescript
// Example only — adapt to your project

// ❌ WRONG — One class doing multiple unrelated things
class OrganizationService {
  async createOrganization(data: unknown) { ... }
  async updateSchedule(data: unknown) { ... }
  async deleteOrganization(id: string) { ... }
}


// ✅ CORRECT — One class per file in a domain-specific folder

// organizations/services/organization-services/create-organization.service.ts
export class CreateOrganizationService {
  async execute(data: CreateOrganizationInput): Promise<Organization> { ... }
}

// organizations/services/organization-services/update-schedule.service.ts
export class UpdateScheduleService {
  async execute(organizationId: string, data: UpdateScheduleInput): Promise<Organization> { ... }
}

// organizations/services/organization-services/delete-organization.service.ts
export class DeleteOrganizationService {
  async execute(organizationId: string): Promise<void> { ... }
}
```

**Service directory example — `authorization` module:**

```
# Example only — adapt to your project

server/modules/(identity)/authorization/services/
├── index.ts
├── membership-services/
│   ├── index.ts
│   ├── invite-member.service.ts
│   ├── deactivate-member.service.ts
│   └── change-role.service.ts
└── role-services/
    ├── index.ts
    ├── create-role.service.ts
    └── update-permissions.service.ts
```

---

## Open/Closed — Strategy Pattern

Variant behaviour is added by subclassing, never by editing existing service code.

```typescript
// Example only — adapt to your project

// server/modules/(operations)/orders/domain/rules.ts

export abstract class PriorityStrategy {
  abstract calculateScore(position: number): number;
}

export class NormalPriority extends PriorityStrategy {
  calculateScore(position: number): number {
    return position;
  }
}

export class UrgentPriority extends PriorityStrategy {
  calculateScore(position: number): number {
    return -1000 + position;
  }
}
```

Services consume the strategy via a parameter — a new priority type requires a new subclass, never an `if/else` chain in the service.

---

## Liskov Substitution

Repository classes must honour the full contract they advertise. If you swap `PrismaOrderRepository` for a `MockOrderRepository` in tests, every method must behave identically in terms of return types and raised exceptions. The mock is a drop-in replacement — no special-casing allowed.

---

## Interface Segregation

Repositories expose only what their consumers actually call. Read-heavy flows (e.g., `search` endpoints) only touch read methods; they never import write operations. If a module's repository grows unwieldy, split it into `prisma-read.repo.ts` and `prisma-write.repo.ts` without changing the callers.

---

## Dependency Rule

All source-code dependencies point inward, toward `domain/`. Nothing inside `domain/` knows about services, repositories, or the HTTP layer.

```
api/handler.ts → services/    → domain/
api/handler.ts → repositories/ → domain/
cache/         → domain/
tasks/         → services/    → domain/
server/core/   ← api/, services/
```

---

# Service Layer — Write Orchestration

Services own all write use-cases. They import repositories directly via normal TypeScript imports — no constructor injection, no interface parameters.

```typescript
// Example only — adapt to your project

// server/modules/(core-domain)/items/services/item-services/create-item.service.ts

import { PrismaItemRepository } from '../../repositories/prisma.repo';
import type { Item } from '../../domain/entities';
import { ItemAlreadyExistsError } from '../../domain/exceptions';

interface CreateItemData {
  name: string;
  description?: string;
  organizationId: string;
}

export class CreateItemService {
  async execute(data: CreateItemData): Promise<Item> {
    const repo = new PrismaItemRepository();

    const existing = await repo.getByName(data.name);
    if (existing) {
      throw new ItemAlreadyExistsError();
    }

    return repo.save(data);
  }
}
```

**Multi-step atomic transaction example:**

```typescript
// Example only — adapt to your project

// server/modules/(identity)/users/services/auth-services/register-organization.service.ts

import { prisma } from '@/server/config/db';
import { PrismaOrganizationRepository } from '../../repositories/prisma.repo';
import { PrismaAuthorizationRepository } from '../../../authorization/repositories/prisma.repo';

export class RegisterOrganizationService {
  /**
   * Organization self-registration — one atomic transaction:
   * 1. Create organization record (with ownerUserId)
   * 2. Seed organization roles (ADMIN, MANAGER, MEMBER)
   * 3. Seed default role permissions
   * 4. Create organization membership for owner → ADMIN role
   */
  async execute(data: RegisterOrganizationInput): Promise<Organization> {
    return prisma.$transaction(async (tx) => {
      const org = await tx.organization.create({ data: { ...data } });
      const adminRole = await tx.role.create({ data: { name: 'ADMIN', organizationId: org.id, isSystem: true } });
      await tx.rolePermission.createMany({ data: defaultAdminPermissions(adminRole.id) });
      await tx.membership.create({ data: { userId: data.ownerUserId, organizationId: org.id, roleId: adminRole.id } });
      return org;
    });
  }
}
```

**Real-world service examples across modules:**

```typescript
// Example only — adapt to your project

// server/modules/(identity)/users/services/auth-services/login.service.ts
export class LoginService {
  async execute(email: string, password: string): Promise<TokenPair> { ... }
}

// server/modules/(identity)/authorization/services/membership-services/invite-member.service.ts
export class InviteMemberService {
  async execute(data: InviteMemberInput): Promise<Membership> { ... }
}
```

**Cross-module reads inside a write service:**

```typescript
// Example only — adapt to your project

// server/modules/(operations)/orders/services/order-services/create-order.service.ts

import { PrismaOrderRepository } from '../../repositories/prisma.repo';
import { PrismaItemRepository } from '../../../(core-domain)/items/repositories/prisma.repo'; // cross-module read
import { ItemNotFoundError } from '../../domain/exceptions';

export class CreateOrderService {
  async execute(data: CreateOrderInput): Promise<Order> {
    const itemRepo = new PrismaItemRepository();
    const item = await itemRepo.getById(data.itemId);
    if (!item) throw new ItemNotFoundError();

    const orderRepo = new PrismaOrderRepository();
    return orderRepo.save(data);
  }
}
```

---

# Repository Layer — Data Access

The repository is the **only place that touches Prisma**. It owns all read and write queries for its module and maps between Prisma models and domain entities. Read methods and write methods are clearly sectioned.

```typescript
// Example only — adapt to your project

// server/modules/(core-domain)/items/repositories/prisma.repo.ts

import { prisma } from '@/server/config/db';
import type { Item as ItemPrisma } from '@prisma/client';
import type { Item } from '../domain/entities';

interface CreateItemData {
  name: string;
  description?: string;
  organizationId: string;
}

export class PrismaItemRepository {

  // ── Reads ─────────────────────────────────────────────────────────────────

  async getById(itemId: string): Promise<Item | null> {
    const row = await prisma.item.findUnique({ where: { id: itemId } });
    return row ? this._mapToDomain(row) : null;
  }

  async getByName(name: string): Promise<Item | null> {
    const row = await prisma.item.findFirst({ where: { name: name.toLowerCase() } });
    return row ? this._mapToDomain(row) : null;
  }

  async listByOrganization(organizationId: string): Promise<Item[]> {
    const rows = await prisma.item.findMany({
      where: { organizationId, isActive: true },
      orderBy: { createdAt: 'desc' },
    });
    return rows.map((r) => this._mapToDomain(r));
  }

  async existsByEmail(email: string): Promise<boolean> {
    const count = await prisma.item.count({ where: { name: email.toLowerCase() } });
    return count > 0;
  }

  // ── Writes ────────────────────────────────────────────────────────────────

  async save(data: CreateItemData): Promise<Item> {
    const row = await prisma.item.create({ data: this._mapToORM(data) });
    return this._mapToDomain(row);
  }

  async update(itemId: string, data: Partial<CreateItemData>): Promise<Item> {
    const row = await prisma.item.update({
      where: { id: itemId },
      data: this._mapToORM(data),
    });
    return this._mapToDomain(row);
  }

  async delete(itemId: string): Promise<void> {
    await prisma.item.delete({ where: { id: itemId } });
  }

  // ── Mappers ───────────────────────────────────────────────────────────────

  private _mapToDomain(row: ItemPrisma): Item {
    return {
      id: row.id,
      name: row.name,
      description: row.description ?? undefined,
      isActive: row.isActive,
      organizationId: row.organizationId,
      createdAt: row.createdAt,
    };
  }

  private _mapToORM(data: Partial<CreateItemData>): Partial<ItemPrisma> {
    return {
      ...(data.name && { name: data.name.toLowerCase() }),
      ...(data.description !== undefined && { description: data.description }),
      ...(data.organizationId && { organizationId: data.organizationId }),
    };
  }
}
```

---

# API Layer — HTTP Controllers

The handler layer is **thin by law**. It delegates immediately — never implements. For reads (GET), it imports and calls the repository directly. For writes (POST/PUT/PATCH/DELETE), it imports and calls the service. All responses use the standardized response helpers.

```typescript
// Example only — adapt to your project

// server/modules/(core-domain)/items/api/handler.ts

import { NextRequest, NextResponse } from 'next/server';
import { withErrorHandler } from '@/server/core/handler-wrapper';
import { requiresPermission } from '@/server/core/permissions';
import { createSuccessResponse, createPaginatedResponse } from '@/server/core/responses';
import { NotFoundError } from '@/server/core/exceptions';
import { PrismaItemRepository } from '../repositories/prisma.repo';
import { CreateItemService } from '../services/item-services/create-item.service';
import { DeleteItemService } from '../services/item-services/delete-item.service';
import { createItemSchema } from './schemas';

// ── GET /api/v1/items ─────────────────────────────────────────────────────────

/**
 * List all active items for an organization.
 * Returns a paginated list filtered by organizationId query param.
 */
export const handleListItems = withErrorHandler(async (req: NextRequest) => {
  const { searchParams } = new URL(req.url);
  const organizationId = searchParams.get('organizationId') ?? '';

  const repo = new PrismaItemRepository();
  const items = await repo.listByOrganization(organizationId);

  return NextResponse.json(
    createPaginatedResponse(items, items.length, 1, 50, 'Items retrieved successfully'),
  );
});

// ── GET /api/v1/items/[id] ────────────────────────────────────────────────────

/**
 * Retrieve a single item by ID.
 * Returns 404 if the item does not exist.
 */
export const handleGetItem = withErrorHandler(
  async (req: NextRequest, ctx: { params: { id: string } }) => {
    const repo = new PrismaItemRepository();
    const item = await repo.getById(ctx.params.id);
    if (!item) throw new NotFoundError('Item not found');

    return NextResponse.json(
      createSuccessResponse(item, 'Item retrieved successfully'),
    );
  },
);

// ── POST /api/v1/items ────────────────────────────────────────────────────────

/**
 * Create a new item within an organization.
 * Validates the payload and delegates creation to the service layer.
 * Requires items:create permission.
 */
export const handleCreateItem = withErrorHandler(
  requiresPermission('items:create')(async (req: NextRequest) => {
    const body = await req.json();
    const payload = createItemSchema.parse(body);

    const service = new CreateItemService();
    const item = await service.execute(payload);

    return NextResponse.json(
      createSuccessResponse(item, 'Item created successfully'),
      { status: 201 },
    );
  }),
);

// ── DELETE /api/v1/items/[id] ─────────────────────────────────────────────────

/**
 * Delete an item by ID.
 * Permanently removes the item. Requires items:delete permission.
 */
export const handleDeleteItem = withErrorHandler(
  requiresPermission('items:delete')(
    async (req: NextRequest, ctx: { params: { id: string } }) => {
      const service = new DeleteItemService();
      await service.execute(ctx.params.id);

      return NextResponse.json(
        createSuccessResponse(null, 'Item deleted successfully'),
      );
    },
  ),
);
```

---

# Connection Bridge: `app/api/` ↔ `server/modules/`

The `app/api/v1/` directory is the Next.js HTTP routing layer. Every `route.ts` file is a thin re-export of the named handler functions from `server/modules/`. No logic whatsoever lives in `route.ts` files.

```typescript
// Example only — adapt to your project

// src/app/api/v1/items/route.ts
import {
  handleListItems,
  handleCreateItem,
} from '@/server/modules/(core-domain)/items/api/handler';

export const GET = handleListItems;
export const POST = handleCreateItem;
```

```typescript
// Example only — adapt to your project

// src/app/api/v1/items/[id]/route.ts
import {
  handleGetItem,
  handleUpdateItem,
  handleDeleteItem,
} from '@/server/modules/(core-domain)/items/api/handler';

export const GET = handleGetItem;
export const PUT = handleUpdateItem;
export const DELETE = handleDeleteItem;
```

> Think of `app/api/v1/**/route.ts` as Django's `urls.py` — pure routing declarations. Think of `server/modules/<module>/api/handler.ts` as Django's `router.py` — the actual controller logic.

---

# RBAC System

> This section governs `server/core/permissions.ts` and the `authorization` module. The permission wrapper touches every API handler in every module, so the contract here is non-negotiable.

## Architecture Decision: Manual Tenant-Scoped RBAC

The platform uses manual RBAC. The rationale:

| Requirement                                                                    | Library Support | Manual |
| ------------------------------------------------------------------------------ | :-------------: | :----: |
| Per-organization role scoping (same user, different roles in different orgs)   | ❌              | ✅     |
| Runtime-configurable permissions (org admin edits permissions via UI)          | ❌              | ✅     |
| Row-level scoping (`orders:own:read`)                                          | ❌              | ✅     |
| No library deprecation risk                                                    | ❌              | ✅     |

---

## `server/core/permissions.ts` — Full Contract

```typescript
// Example only — adapt to your project

// src/server/core/permissions.ts

import { NextRequest, NextResponse } from 'next/server';
import { PrismaAuthorizationRepository } from '@/server/modules/(identity)/authorization/repositories/prisma.repo';
import { createErrorResponse } from './responses';

type RouteHandler<C = unknown> = (req: NextRequest, ctx?: C) => Promise<NextResponse>;

// ─── Core Resolver ────────────────────────────────────────────────────────────

/**
 * Single resolver — everything flows through here.
 * Returns the full permission set for a user in an organization.
 * Live DB query — cache is handled inside the authorization repo.
 */
export async function getPermissions(
  userId: string,
  organizationId: string,
): Promise<Set<string>> {
  const repo = new PrismaAuthorizationRepository();
  const perms = await repo.getPermissions(userId, organizationId);
  return new Set(perms);
}

export async function hasPermission(
  userId: string,
  organizationId: string,
  permission: string,
): Promise<boolean> {
  const perms = await getPermissions(userId, organizationId);
  return perms.has(permission);
}

// ─── Next.js Handler Wrapper ─────────────────────────────────────────────────

/**
 * Wrapper for Next.js Route Handler functions.
 * Usage: requiresPermission('orders:create')(handlerFn)
 * Resolves organizationId from path params first, then query string.
 */
export function requiresPermission(permission: string) {
  return <C extends { params?: Record<string, string> }>(
    handler: RouteHandler<C>,
  ): RouteHandler<C> => {
    return async (req: NextRequest, ctx?: C) => {
      const userId = req.headers.get('x-user-id');
      const organizationId =
        ctx?.params?.organizationId ??
        new URL(req.url).searchParams.get('organizationId');

      if (!userId || !organizationId) {
        return NextResponse.json(
          createErrorResponse('Unauthorized'),
          { status: 401 },
        );
      }

      const allowed = await hasPermission(userId, organizationId, permission);
      if (!allowed) {
        return NextResponse.json(
          createErrorResponse(`Permission required: ${permission}`),
          { status: 403 },
        );
      }

      return handler(req, ctx);
    };
  };
}
```

---

## Permission String Convention

All permission strings use explicit `resource:action` format. No wildcards. No `manage` shorthand — always expand to discrete verbs.

```
# Core domain
items:create       items:update       items:delete       items:read

# Operations
orders:write       orders:read        orders:own:read

# Organization management
organization:settings:write   organization:settings:read

# RBAC
members:invite      members:deactivate     members:read
roles:create        roles:update           roles:delete
```

---

## Cache Invalidation Contract

Any service that mutates `memberships` or `rolePermissions` **must** call the cache invalidation helper **after** the DB transaction commits. Never inside `prisma.$transaction()`.

```typescript
// Example only — adapt to your project

// server/modules/(identity)/authorization/services/membership-services/invite-member.service.ts

import { prisma } from '@/server/config/db';
import { PrismaAuthorizationRepository } from '../../repositories/prisma.repo';
import { AuthorizationCacheRepo } from '../../cache/redis.repo';

export class InviteMemberService {
  async execute(data: InviteMemberInput): Promise<Membership> {
    const repo = new PrismaAuthorizationRepository();
    const cache = new AuthorizationCacheRepo();

    const membership = await prisma.$transaction(async (tx) => {
      return repo.createMembership(data, tx);
    });

    // Cache invalidation AFTER commit — never inside the transaction
    await cache.invalidate(data.organizationId, data.userId);
    return membership;
  }
}
```

---

## Row-Level Scoping (`orders:own:read`)

When a user holds only `orders:own:read`, the handler applies an additional filter before calling the repository.

```typescript
// Example only — adapt to your project

// server/modules/(operations)/orders/api/handler.ts

/**
 * List orders for an organization.
 * Users with orders:read see all orders. Users with orders:own:read see only their own.
 */
export const handleListOrders = withErrorHandler(async (req: NextRequest) => {
  const { searchParams } = new URL(req.url);
  const organizationId = searchParams.get('organizationId') ?? '';
  const userId = req.headers.get('x-user-id') ?? '';

  const perms = await getPermissions(userId, organizationId);
  const repo = new PrismaOrderRepository();

  if (perms.has('orders:read')) {
    const orders = await repo.listByOrganization(organizationId);
    return NextResponse.json(createPaginatedResponse(orders, orders.length, 1, 50));
  }

  if (perms.has('orders:own:read')) {
    const authRepo = new PrismaAuthorizationRepository();
    const assigneeId = await authRepo.getAssigneeByUser(userId, organizationId);
    const orders = assigneeId ? await repo.listByAssignee(organizationId, assigneeId) : [];
    return NextResponse.json(createPaginatedResponse(orders, orders.length, 1, 50));
  }

  return NextResponse.json(createErrorResponse('Permission required'), { status: 403 });
});
```

---

# Real-Time State Layer (Redis)

> This section applies only to modules that maintain live, distributed state. Purely CRUD modules (`items`, `organizations`, `users`, `authorization`) do **not** need a `cache/` folder.

## Deciding If Your Module Needs Redis

| Module needs Redis? | Criteria                                                                     |
| ------------------- | ---------------------------------------------------------------------------- |
| ✅ Yes              | Live counters, priority queues, real-time availability, pub/sub, TTL lookups |
| ❌ No               | Simple CRUD, data that only changes on explicit user action                  |

| Module          | Needs Redis? | Reason                                                            |
| --------------- | :---------: | ----------------------------------------------------------------- |
| `orders`        | ❌          | Booking uses optimistic concurrency control on DB level           |
| `scheduling`    | ⚠️          | Cache exists but is disabled — preserved for future use           |
| `search`        | ✅          | Redis search index for multi-organization discovery               |
| `authorization` | ✅          | Permission cache (TTL 300s, key: `perm:{orgId}:{userId}`)         |
| `organizations` | ❌          | Pure CRUD                                                         |
| `users`         | ❌          | Pure CRUD                                                         |
| `catalog`       | ❌          | CRUD                                                              |

---

## Redis Repository Example

```typescript
// Example only — adapt to your project

// server/modules/(identity)/authorization/cache/redis.repo.ts

import { redis } from '@/server/config/redis';

export class AuthorizationCacheRepo {
  private readonly TTL = 300; // 5 minutes

  private _key(organizationId: string, userId: string): string {
    return `perm:${organizationId}:${userId}`;
  }

  async getCachedPermissions(
    organizationId: string,
    userId: string,
  ): Promise<string[] | null> {
    const cached = await redis.get(this._key(organizationId, userId));
    return cached ? (JSON.parse(cached) as string[]) : null;
  }

  async setCachedPermissions(
    organizationId: string,
    userId: string,
    permissions: string[],
  ): Promise<void> {
    await redis.set(
      this._key(organizationId, userId),
      JSON.stringify(permissions),
      'EX',
      this.TTL,
    );
  }

  async invalidate(organizationId: string, userId: string): Promise<void> {
    await redis.del(this._key(organizationId, userId));
  }
}
```

---

## Redis Failure Recovery

- Every Redis write must have a corresponding database write first. The SQL database is the source of truth.
- A management script must exist to rebuild Redis state from the DB on reconnect.
- The rebuild script must be idempotent — safe to run multiple times.
- Never use `redis.keys('*')` in production — use `SCAN` + `DEL` in batches.

---

# WebSocket + Pub/Sub Rules

## Event Flow (Always in this order)

```
Service → DB write → Redis state update → Redis PUBLISH → WS consumer → client push
```

Never skip steps. The DB write always precedes the Redis write.

## Consumer Rules

- Consumers only subscribe and forward. **Zero business logic in consumers.**
- Consumers must handle disconnections gracefully — must never crash the server.
- Channel names are entity-scoped:

```typescript
// Example only — adapt to your project

`queue:global`           // queue module — all queue events
`item:${id}`             // item-scoped events
`order:${id}`            // order-scoped events
`organization:${id}`     // organization-scoped events
```

---

# Cross-Module Communication

When a service or handler needs data from another module, it imports that module's `prisma.repo.ts` directly. No injected interfaces, no adapter layers — just a plain import.

```typescript
// Example only — adapt to your project

// ✅ CORRECT — service reads from another module via direct repository import
// server/modules/(operations)/orders/services/order-services/create-order.service.ts

import { PrismaOrderRepository } from '../../repositories/prisma.repo';
import { PrismaItemRepository } from '../../../(core-domain)/items/repositories/prisma.repo';
import { PrismaAuthorizationRepository } from '../../../(identity)/authorization/repositories/prisma.repo';

export class CreateOrderService {
  async execute(data: CreateOrderInput): Promise<Order> {
    const item = await new PrismaItemRepository().getById(data.itemId);
    if (!item) throw new ItemNotFoundError();

    return new PrismaOrderRepository().save(data);
  }
}


// ❌ WRONG — importing another module's Prisma model directly
import { prisma } from '@/server/config/db';
const item = await prisma.item.findUnique(...);  // Forbidden outside items/repositories/

// ❌ WRONG — importing another module's repository from inside domain/
// domain/ must never import from other modules
import { PrismaAuthorizationRepository } from '../../../(identity)/authorization/repositories/prisma.repo';
```

---

# Dependency Flow Matrix

```
                  domain   services   repositories   api/handler   tasks   cache   server/core
domain              ✅        ❌           ❌              ❌          ❌      ❌         ❌
services            ✅        ❌           ✅              ❌          ❌      ✅         ✅
repositories        ✅        ❌           ❌              ❌          ❌      ❌         ❌
api/handler         ❌        ✅           ✅              ❌          ❌      ❌         ✅
tasks               ❌        ✅           ❌              ❌          ❌      ❌         ❌
cache               ✅        ❌           ❌              ❌          ❌      ❌         ❌
server/core         ❌        ❌           ❌              ❌          ❌      ❌         ✅
```

> Cross-module: services and handlers may import repositories from other modules via direct imports. The `domain/` layer never imports from any other module.

---

# Automated Enforcement (ESLint)

Use `eslint-plugin-boundaries` to enforce layer contracts at lint time.

```javascript
// Example only — adapt to your project

// eslint.config.mjs
import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'server-domain',       pattern: 'src/server/modules/*/domain/**' },
        { type: 'server-services',     pattern: 'src/server/modules/*/services/**' },
        { type: 'server-repositories', pattern: 'src/server/modules/*/repositories/**' },
        { type: 'server-api',          pattern: 'src/server/modules/*/api/**' },
        { type: 'server-cache',        pattern: 'src/server/modules/*/cache/**' },
        { type: 'server-tasks',        pattern: 'src/server/modules/*/tasks/**' },
        { type: 'server-core',         pattern: 'src/server/core/**' },
        { type: 'client-shared',       pattern: 'src/shared/**' },
        { type: 'client-modules',      pattern: 'src/modules/**' },
        { type: 'app-api',             pattern: 'src/app/api/**' },
      ],
    },
    rules: {
      // Domain layer is fully independent
      'boundaries/element-types': ['error', {
        default: 'disallow',
        rules: [
          { from: 'server-domain',       allow: [] },
          { from: 'server-services',     allow: ['server-domain', 'server-repositories', 'server-cache', 'server-core'] },
          { from: 'server-repositories', allow: ['server-domain'] },
          { from: 'server-api',          allow: ['server-services', 'server-repositories', 'server-core'] },
          { from: 'server-tasks',        allow: ['server-services'] },
          { from: 'server-cache',        allow: ['server-domain'] },
          { from: 'app-api',             allow: ['server-api'] }, // route.ts → handler.ts only
        ],
      }],
    },
  },
];
```

---

# Server-Side Architectural Rules

- **Rule 1 (Domain Purity):** The `domain/` layer must stay 100% pure: no Prisma imports, no Redis, no BullMQ, no Next.js, no HTTP concerns. Tests in `domain/` must pass with zero external dependencies.

- **Rule 2 (Thin API Layer):** `handler.ts` contains zero business logic. For reads: import and call the repository directly. For writes: import and call the service. No orchestration beyond that single delegation.

- **Rule 3 (Thin Route Files):** `app/api/v1/**/route.ts` files contain only re-exports of handler functions. They are forbidden from importing anything except `handler.ts`.

- **Rule 4 (Services Own All Writes):** Every POST/PUT/PATCH/DELETE must go through a service class with an `execute()` method. Never write to the DB directly from `handler.ts`.

- **Rule 5 (Repositories Own Prisma Access):** Direct `prisma.*` calls are strictly forbidden everywhere except `repositories/prisma.repo.ts`. Any layer that needs data goes through the repository.

- **Rule 6 (One Service, One Use-Case):** A service class has exactly one public method (`execute()`) performing exactly one use-case. If two actions are needed, that is two service classes in two files.

- **Rule 7 (No Constructor Injection):** Services and repositories are never passed as parameters to `constructor`. All dependencies are imported directly at the top of the file and instantiated at call time. No DI containers. No TypeScript `interface` protocol injection.

- **Rule 8 (Thin Tasks):** BullMQ task files are thin shims — they instantiate a service and call `execute()`. Maximum 15 lines per task file. All logic lives in the service.

- **Rule 9 (Exception Hierarchy):** All domain exceptions extend `server/core/exceptions.ApplicationError`. Services throw domain exceptions. Handlers catch them via `withErrorHandler()`.

- **Rule 10 (Redis After DB):** Every Redis write must be preceded by a successful database write. The SQL database is the source of truth. Redis is the speed layer. They are never out of sync for more than one operation.

- **Rule 11 (Cache Invalidation Outside Transactions):** Never call cache invalidation functions inside `prisma.$transaction()`. Call them after the transaction resolves.

- **Rule 12 (No Cross-Module Prisma Imports):** When a service or handler needs data from another module, it imports and uses that module's `prisma.repo.ts`. Direct use of `prisma.*` for another module's data is forbidden everywhere except within that module's own repository.

- **Rule 13 (Permission Resolution via Core):** All permission checks must flow through `server/core/permissions.ts`. No service, repository, or handler may directly query membership or role permission tables for authorization purposes.

- **Rule 14 (Explicit Permission Strings):** Permission strings are explicit `resource:action` tuples. No wildcards. No `manage` shorthand — expand to `create`, `update`, `delete` individually.

- **Rule 15 (System Roles Are Immutable):** System roles (`isSystem: true`) may not be deleted or have their permissions modified. Enforce this as a domain rule in `authorization/domain/rules.ts`.

- **Rule 16 (Organization Must Have One Admin):** An organization must always have at least one active ADMIN member. `DeactivateMemberService` and `ChangeRoleService` must verify this invariant before committing.

- **Rule 17 (No Naming Dumping Grounds):** Folder names like `misc/`, `helpers/`, `common/`, or `other/` are strictly banned. Every folder must have a clear, bounded responsibility.

- **Rule 18 (Encapsulate Business Conditionals):** Never write loose boolean conditions inside handlers or services (e.g., `user.role === 'ADMIN' && org.isActive`). Wrap them in descriptive functions in `domain/rules.ts` (e.g., `canManageOrganization(user, org)`).

- **Rule 19 (Safe Deletability):** If dropping a module's folder breaks an unrelated module, your domain boundaries are bleeding. Each module must be independently removable.

- **Rule 20 (No SCAN-less Redis Deletes in Production):** Use `SCAN` + `DEL` in batches. `redis.keys('*')` blocks the event loop and is forbidden in any environment with real data.

- **Rule 21 (Search Module Is Read-Only):** The `search` module contains no services and no write repository methods. It reads from its search index only.

- **Rule 22 (API Documentation Is Mandatory):** Every handler function must have a JSDoc comment, a Zod-validated input, and a typed response. Missing documentation is a bug, not a nice-to-have.

- **Rule 23 (Standardized API Responses):** All handlers must return responses via `createSuccessResponse()`, `createPaginatedResponse()`, or let `withErrorHandler()` format errors via `createErrorResponse()`. Never return raw data or ad-hoc JSON structures.

- **Rule 24 (Frontend ↔ Server Contract via Shared Types):** Zod schemas exported from `server/modules/<module>/api/schemas.ts` are the source of truth for the frontend `api/contracts/` and `api/mappers/` layers. Never duplicate type definitions between frontend and server.

---

# Anti-Patterns Reference

| If you see this                                               | Module context | The fix                                                                                          |
| ------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------ |
| `prisma.item.findUnique()` inside a service                   | any            | Move to `PrismaItemRepository`; import the repo class in the service                            |
| Business logic in `handler.ts`                                | any            | Extract to a service class; handler calls `service.execute()`                                   |
| `handler.ts` writing to DB directly via `prisma.*`            | any            | Create a service class — handler never writes                                                    |
| Logic inside `app/api/**/route.ts`                            | any            | `route.ts` is re-exports only; move logic to `server/modules/<module>/api/handler.ts`            |
| `class BaseXxxService { constructor(repo) { ... } }`          | any            | Remove constructor injection; services import repos directly at the top of the file              |
| Repository passed as constructor parameter                    | any            | Import the concrete repository class directly — no constructor injection                         |
| One service file handling two unrelated use-cases             | any            | Split into two files, one per use-case                                                           |
| Strategy behaviour hardcoded with `if/else` in a service      | any            | Create a strategy subclass in `domain/rules.ts`                                                  |
| Permission check duplicated in service AND handler            | any            | Checks live only in `server/core/permissions.ts` via `requiresPermission()` wrapper             |
| `import { prisma } from ... ` inside `orders/` for item data  | orders         | Import `PrismaItemRepository` and use its read methods                                           |
| Cache invalidation inside `prisma.$transaction()`             | any            | Move after the transaction resolves                                                              |
| Handler returning raw Prisma object without mapping           | any            | Map through `_mapToDomain()` in the repository before returning                                  |
| `search` module importing a write repository or service       | search         | The `search` module is read-only by Rule 21                                                      |
| Missing JSDoc on a handler function                           | any            | Add JSDoc comment with summary, params, and return type per Rule 22                              |
| Handler returning `NextResponse.json({ id, name, ... })`      | any            | Wrap in `createSuccessResponse(item, 'Item retrieved successfully')` per Rule 23                 |
| Zod types duplicated in `api/contracts/` and `server/schemas` | any            | Export from `server/modules/<module>/api/schemas.ts` and import in frontend `api/contracts/`     |
| `redis.keys('search:*')` in production                        | search         | Use `redis.scan()` + `DEL` in batches per Rule 20                                               |
| Domain exception without extending `ApplicationError`         | any            | All domain exceptions must extend `server/core/exceptions.ApplicationError`                      |
| `withErrorHandler` omitted on a handler function              | any            | All exported handler functions must be wrapped with `withErrorHandler()`                         |

---

# Server-Side Final Architectural Performance Metrics

| Area                        | Result    | Target Metric Met                                                              |
| --------------------------- | --------- | ------------------------------------------------------------------------------ |
| **Scalability**             | High      | Unlocked parallel workflows for multi-team execution.                          |
| **Maintainability**         | High      | Code changes stay contained inside module directories.                         |
| **Team Collaboration**      | Excellent | Minimized codebase conflicts via clear domain boundaries.                      |
| **Domain Isolation**        | Strong    | Core business rules are protected from ORM and HTTP churn.                     |
| **Refactoring Safety**      | High      | Isolated scopes make code deletion safe and reliable.                          |
| **Feature Ownership**       | Clear     | Modules track perfectly to business domains and engineering teams.             |
| **Testing**                 | Easier    | Pure domain layers accept unit testing without any Next.js or Prisma setup.    |
| **Cognitive Load**          | Lower     | Developers only need to reason about one module directory at a time.           |
| **Microservice Readiness**  | Good      | Modules are prepared for future decomposition into standalone services.        |
| **Simplicity**              | Excellent | No DI containers, no injected interfaces — direct imports reduce mental overhead without sacrificing layer clarity. |
| **API Discoverability**     | Excellent | JSDoc + Zod schemas enforced on every handler, ensuring documentation is always accurate. |
| **Client Integration**      | Excellent | Standardized response structures and shared Zod types eliminate special-casing on the frontend, reducing integration bugs. |
