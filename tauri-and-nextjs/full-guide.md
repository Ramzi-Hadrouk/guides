# Tauri + Next.js Desktop Application Architecture Guide
# (Scalable Enterprise Version)

> **Disclaimer:** All code snippets, module names, struct names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions. Nothing here is prescriptive for a particular industry or application.

---

This architecture is designed for:

- Cross-platform desktop applications built with **Tauri v2 + Next.js**
- Multi-team development on larger desktop products
- **Tauri (Rust) as the single source of truth** — all shared types are generated from Rust structs and consumed by the frontend; no type duplication between layers
- Redux Toolkit for client-side state + Tauri IPC for all data operations
- SQLite via `sqlx` for local persistent storage
- Long-term maintainability and safe refactoring
- Feature isolation on the frontend
- Domain-driven design in the Rust backend
- Strict separation between UI and desktop backend concerns

The real goals are **clear ownership**, **end-to-end type safety**, **scalability**, **strict boundaries**, **low coupling**, **easier refactoring**, **safe deletion**, **team scalability**, and **localized UI ownership per module**.

---

# Full Project Structure

```txt
.
├── .env
├── .env.example
├── next.config.ts                        # output: 'export', distDir: 'out'
├── package.json
├── tsconfig.json
│
│  ╔════════════════════════════════════════════════════════════════╗
│  ║  TAURI BACKEND  (Rust — Single Source of Truth for Types)     ║
│  ╚════════════════════════════════════════════════════════════════╝
│
├── src-tauri/
│   ├── src/
│   │   ├── main.rs                       # Binary entry point
│   │   ├── lib.rs                        # App builder: registers all commands, state, plugins
│   │   │
│   │   ├── modules/                      # Domain modules (mirror of server/modules/)
│   │   │   ├── mod.rs
│   │   │   │
│   │   │   │  # ─── Domain Group: Identity & Access ─────────────────────────
│   │   │   ├── identity/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── users/
│   │   │   │   └── authorization/
│   │   │   │
│   │   │   │  # ─── Domain Group: Core Domain ──────────────────────────────
│   │   │   ├── core_domain/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── items/
│   │   │   │   └── customers/
│   │   │   │
│   │   │   │  # ─── Domain Group: Operations ────────────────────────────────
│   │   │   ├── operations/
│   │   │   │   ├── mod.rs
│   │   │   │   └── orders/
│   │   │   │
│   │   │   │  # ─── Domain Group: Integrations ─────────────────────────────
│   │   │   ├── integrations/
│   │   │   │   ├── mod.rs
│   │   │   │   └── external_services/
│   │   │   │
│   │   │   └── <module>/                 # Template — every module follows this layout
│   │   │       ├── mod.rs               # pub use; re-exports commands for lib.rs
│   │   │       ├── commands.rs          # Tauri IPC commands — thin delegation ONLY
│   │   │       ├── domain/
│   │   │       │   ├── mod.rs
│   │   │       │   ├── entities.rs      # Structs with #[derive(Serialize, Deserialize, TS)]
│   │   │       │   ├── value_objects.rs # Immutable, self-validating types
│   │   │       │   ├── errors.rs        # Domain-specific error variants
│   │   │       │   └── rules.rs         # Pure business rule functions
│   │   │       ├── services/
│   │   │       │   ├── mod.rs
│   │   │       │   └── <verb>_<entity>.rs  # One file per use-case
│   │   │       ├── repository.rs        # sqlx SQLite data-access — reads and writes
│   │   │       └── tests/
│   │   │           ├── unit/            # Domain + service unit tests (no DB)
│   │   │           └── integration/     # Command + DB integration tests
│   │   │
│   │   ├── core/
│   │   │   ├── mod.rs
│   │   │   ├── error.rs                 # AppError hierarchy + IPC serialization
│   │   │   ├── response.rs              # IpcResponse<T> + PaginatedData<T> with #[derive(TS)]
│   │   │   ├── permissions.rs           # RBAC resolver and require_permission guard
│   │   │   └── pagination.rs            # Shared pagination helpers
│   │   │
│   │   └── config/
│   │       ├── mod.rs
│   │       ├── database.rs              # SqlitePool singleton + migration runner
│   │       └── app.rs                   # App settings and environment config
│   │
│   ├── bindings/                        # Auto-generated TypeScript types (ts-rs output)
│   │   ├── Item.ts
│   │   ├── CreateItemInput.ts
│   │   ├── IpcResponse.ts
│   │   ├── PaginatedData.ts
│   │   └── ...                          # One .ts file per #[derive(TS)] struct
│   │
│   ├── migrations/                      # sqlx SQL migration files
│   │   └── 20240101000000_init.sql
│   │
│   ├── Cargo.toml
│   ├── build.rs                         # Triggers ts-rs type export at build time
│   └── tauri.conf.json
│
│  ╔════════════════════════════════════════════════════════════════╗
│  ║  NEXT.JS FRONTEND  (Pages + Client-Side Architecture)         ║
│  ╚════════════════════════════════════════════════════════════════╝
│
└── src/
    ├── app/                             # Next.js App Router — PAGES ONLY
    │   ├── (dashboard)/                 # Dashboard page routes
    │   │   └── ...
    │   ├── (auth)/                      # Auth page routes
    │   │   └── ...
    │   ├── (public)/                    # Public page routes
    │   │   └── ...
    │   └── layout.tsx                   # Root layout
    │
    └── frontend/                        # All frontend application code lives here
        │
        ├── bootstrap/                   # App startup logic before first render
        │
        ├── core/                        # Infrastructure layer
        │   ├── ipc/                     # Tauri IPC infrastructure (replaces api/)
        │   │   ├── base/                # invoke() wrapper, error normalizer, retry
        │   │   ├── contracts/           # Per-module typed command function wrappers
        │   │   ├── mappers/             # IPC response → domain model converters
        │   │   └── index.ts
        │   ├── auth/                    # Auth infrastructure
        │   ├── permissions/             # Client-side RBAC engine
        │   ├── storage/                 # Local storage abstractions
        │   ├── logging/                 # Logging infrastructure
        │   └── monitoring/              # Error tracking
        │
        ├── shared/                      # Globally reusable, business-agnostic code
        │   ├── ui/                      # Generic UI components (Button, Input, Modal...)
        │   ├── hooks/                   # Generic utility hooks (useDebounce, useToggle...)
        │   ├── utils/                   # Pure utility functions (date, currency, format...)
        │   ├── lib/                     # Third-party library wrappers
        │   ├── constants/               # Global application constants
        │   ├── types/
        │   │   └── generated/           # Symlink or copy of src-tauri/bindings/ (ts-rs output)
        │   ├── icons/
        │   ├── assets/
        │   ├── i18n/                    # Global i18n config and utils
        │   └── layouts/                 # All shared application shell layouts
        │       ├── (dashboard)/
        │       │   └── dashboard-layout/
        │       │       ├── components/
        │       │       ├── hooks/
        │       │       ├── types/
        │       │       └── index.ts
        │       ├── (auth)/
        │       │   └── auth-layout/
        │       ├── (public)/
        │       │   └── public-layout/
        │       └── index.ts
        │
        ├── modules/                     # Business domain modules
        │   ├── (dashboard)/
        │   │   ├── items/
        │   │   ├── orders/
        │   │   └── team/
        │   ├── (auth)/
        │   │   └── login/
        │   └── (public)/
        │       └── landing/
        │
        ├── routes/                      # Route definitions and navigation helpers
        ├── store/                       # Redux Toolkit slices and global state
        └── styles/                      # Global styles, themes, design tokens
```

> **Environment Files:** `.env` holds actual configuration values. `.env.example` must be kept in sync whenever keys are added or removed, but only placeholder (non-sensitive) values should be committed.

> **Next.js Static Export:** This is a desktop application. `next.config.ts` must set `output: 'export'` and `distDir: 'out'`. Tauri's `tauri.conf.json` sets `devUrl: "http://localhost:3000"` for development and `frontendDist: "../out"` for production builds. There are **no** Next.js API routes, no server-side rendering, and no server actions in this architecture.

---

---

# PART 1 — FRONTEND ARCHITECTURE

> This section covers the complete `src/frontend/` directory. All content in Part 1 applies strictly to the client-side layer.

---

# Frontend Architecture (Scalable Enterprise Version)

This architecture is designed for:

- Cross-platform desktop applications with complex UI
- Multi-team development
- React / Next.js (static export mode)
- Redux Toolkit for client-side state
- Tauri IPC (`invoke()`) replacing all network calls
- Long-term maintainability
- Feature isolation
- Domain-driven frontend systems

---

# Frontend Project Structure

```txt
src/
├── app/                             # Next.js App Router (routing shell — PAGES ONLY)
│   ├── (dashboard)/
│   ├── (auth)/
│   ├── (public)/
│   └── layout.tsx
│
└── frontend/                        # All frontend code lives here
    │
    ├── bootstrap/
    │
    ├── core/
    │   ├── ipc/
    │   │   ├── base/
    │   │   ├── contracts/
    │   │   ├── mappers/
    │   │   └── index.ts
    │   ├── auth/
    │   ├── permissions/
    │   ├── storage/
    │   └── logging/
    │
    ├── shared/
    │   ├── ui/
    │   ├── hooks/
    │   ├── utils/
    │   ├── lib/
    │   ├── constants/
    │   ├── types/
    │   │   └── generated/            # Auto-generated types from Rust (ts-rs)
    │   ├── icons/
    │   ├── assets/
    │   ├── i18n/
    │   └── layouts/                  # All shared layout shells live here
    │       ├── (dashboard)/
    │       │   └── dashboard-layout/
    │       │       ├── components/
    │       │       ├── hooks/
    │       │       ├── types/
    │       │       └── index.ts
    │       ├── (auth)/
    │       │   └── auth-layout/
    │       ├── (public)/
    │       │   └── public-layout/
    │       └── index.ts
    │
    ├── modules/
    │   ├── (dashboard)/
    │   │   ├── items/
    │   │   │   ├── i18n/
    │   │   │   ├── pages/
    │   │   │   ├── domain/
    │   │   │   ├── shared/
    │   │   │   ├── feature-1/
    │   │   │   │   ├── domain/
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
    │   │   │   ├── feature-2/
    │   │   │   └── index.ts
    │   │   └── ...
    │   ├── (auth)/
    │   └── (public)/
    │
    ├── routes/
    ├── store/
    └── styles/
```

---

# Group Folders

Group folders are folders wrapped in parentheses, for example `(dashboard)`, `(auth)`, `(public)`.

They are used to **group related code without affecting the runtime structure**. The concept applies equally to pages, layouts, modules, and any other folder that benefits from explicit ownership boundaries.

### Why use group folders

- Keep related code together without leaking it into the public path structure
- Separate dashboard, public, and auth concerns cleanly
- Group layouts, modules, and features by business area
- Make ownership clearer for teams
- Reduce cognitive load by making bounded contexts visible in the folder name

### Example intent

```txt
src/frontend/
├── modules/
│   ├── (dashboard)/
│   │   ├── items/
│   │   ├── orders/
│   │   └── team/
│   └── (public)/
│       └── landing/
│
└── shared/
    └── layouts/
        ├── (dashboard)/
        ├── (auth)/
        └── (public)/
```

---

# Frontend Layer Responsibilities

## Root & Infrastructure Layers

| Folder / Layer         | Description & Responsibilities                                                         | Allowed / Contains                                                                                              | Forbidden                                                            |
| ---------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `app/`                 | Next.js routing shell. Page composition and route groups only.                          | Page components, route groups, root layout, loading and error boundaries.                                       | Business logic, feature workflows, domain rules, feature state.      |
| `frontend/bootstrap/`  | Application startup logic executed before first render.                                 | Tauri readiness checks, feature flag init, startup config, service initialization, polyfills.                   | UI components, application layouts, React hooks.                     |
| `frontend/core/`       | Infrastructure layer providing the system's technical, business-agnostic foundation.    | IPC client, auth infrastructure, permissions engine, logging, monitoring, storage abstractions.                  | UI presentation code, business-specific models, feature execution.   |
| `core/ipc/base/`       | Low-level IPC infrastructure wrapping Tauri's `invoke()`.                               | `invoke()` wrappers, error normalization, retry strategies, response interceptors.                               | Business rules, feature-specific logic.                              |
| `core/ipc/contracts/`  | Frontend-facing IPC command abstractions — one file per backend module.                 | Typed command function wrappers (`itemsCommands`, `ordersCommands`), normalized response shapes.                 | UI-specific logic, React hooks, business rules.                      |
| `core/ipc/mappers/`    | Transformation layer converting IPC responses to frontend domain models.                | `mapItemResponseToItem()`, `mapOrderResponse()`, IPC ↔ domain model converters.                                | React primitives, global state interactions.                         |
| `shared/layouts/`      | Application shell layer. All layout shells live here inside `shared/`.                  | Dashboard layout, auth layout, public layout, navigation shells, page containers.                                | Business workflows, IPC calls, thick state management.               |

---

## Global & Domain Shared Layers

| Folder / Layer      | Description & Responsibilities                                                      | Allowed / Contains                                                                                             | Forbidden                                                                    |
| ------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `shared/`           | Purely technical global reusable items completely stripped of business awareness.    | Generic components (`shared/ui/Button`), `useDebounce`, pure utilities (date, currency), global config.         | Domain-aware items (`useItemData`, `ItemCard`), business constraints.         |
| `shared/types/generated/` | Auto-generated TypeScript types from Rust via `ts-rs`. **Must remain immutable.** | Rust-generated interfaces and types. Never edit manually.                                                      | Manual modifications, custom business logic, UI-specific props.              |
| `modules/`          | Business domain layer. Each subfolder is a fully isolated bounded context.          | Independent folders per domain topic (items, orders, team…).                                                    | Cross-module direct file dependencies without going through public APIs.      |
| `module/i18n/`      | Module-level localization files shared across multiple feature folders.             | Shared page titles, shared labels, notification templates, module-wide validation strings.                      | Feature-exclusive strings, domain-agnostic generic strings.                  |
| `module/pages/`     | High-level page layout configuration and feature composition.                       | Layout compositions, route composition, page orchestration.                                                     | Directly handling IPC calls, complicated data orchestration.                 |
| `module/domain/`    | Pure, framework-independent business core logic.                                    | Entities, core business rules, domain services, pure transformations.                                           | React components, Redux logic, IPC invocations, browser API dependencies.    |
| `module/shared/`    | Reusable utilities constrained strictly within this specific business module.       | Shared UI components, hooks, and constants unique to this module's features.                                    | Infrastructure wrappers, globally generic components.                        |

---

## Feature-Level Layers (`modules/<group>/<module>/feature-x/`)

| Folder / Layer         | Description & Responsibilities                                                                | Allowed / Contains                                                                       | Forbidden                                                                            |
| ---------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `feature/domain/`      | Feature-local domain and business rules for this isolated use case.                           | Feature-specific rules, validations, domain-scoped calculations.                          | React context, hooks, global actions.                                                |
| `feature/i18n/`        | Micro-localization files exclusive to this particular use case.                               | Feature-specific button labels, alert messages, inline helper copy.                       | Shared global or module-level strings.                                               |
| `feature/components/`  | Granular UI building blocks isolated inside the local feature space.                          | Small presentational structures, form inputs, simple layout parts.                        | IPC calls, global state synchronization.                                             |
| `feature/sections/`    | Large UI structures orchestrating presentational elements.                                    | `FormSection`, `TableSection`, layout assembly components.                                | Complex domain logic, root framework integrations.                                   |
| `feature/application/` | Workflow coordinator and business orchestration engine.                                       | Business workflows, form submit orchestration, modal toggles, optimistic state updates.   | Low-level HTML structure, primitive rendering.                                       |
| `feature/state/`       | Feature-specific client-side state managers.                                                  | Redux slices, Zustand stores, selectors, local derived metrics.                           | Duplicating data already managed via IPC results.                                    |
| `feature/hooks/`       | Hooks scoping behaviors to this single application scope.                                     | `useFeaturePermissions`, `useFeatureFilters`, `useItemActions`.                           | Globally generic utility functions.                                                  |
| `feature/utils/`       | Functional stateless helpers serving local components.                                        | Deterministic, side-effect-free data manipulation helpers.                                | Managing state, browser mutations.                                                   |
| `feature/validation/`  | Contract constraints evaluating runtime accuracy before processing actions.                   | Zod configurations, Yup schemas, form checking workflows.                                 | Shared cross-domain schemas.                                                         |
| `feature/types/`       | Data definitions internal to this feature.                                                    | Component props types, local state payload structures.                                    | Sharing outside feature boundaries without elevating to module scope.                |
| `feature/constants/`   | Hardcoded parameters defining behavioral elements inside this component.                      | Action configurations, static lists, grid layout configurations.                          | Global system parameters.                                                            |
| `feature/tests/`       | Continuous verification suite for this unit.                                                  | Unit tests, component integration workflows.                                              | Cross-module logic testing.                                                          |
| `feature/index.ts`     | Public API boundary exposing designated capabilities outward.                                 | Explicit named exports defining what other folders can consume.                           | `export * from './everything'`.                                                      |

---

# Frontend Architectural Rules

- **Rule 1 (No Reverse Imports):** Cross-feature direct imports are strictly forbidden. `feature-1` must never import directly from `feature-2`.
- **Rule 2 (Feature Isolation):** Features remain fully autonomous. Share code only by bubbling it up to `module/shared/`, `module/domain/`, or via orchestration patterns.
- **Rule 3 (Strict Downward Flow):** `shared/` must remain completely business-agnostic and never import from `modules/` or `core/`.
- **Rule 4 (Domain Purity):** The `domain/` layer must stay 100% pure — no React components, no hooks, no state managers, no IPC calls.
- **Rule 5 (Thin View Layers):** Pages and components keep business logic out of their views. JSX files focus strictly on presentational mapping.
- **Rule 6 (No Manual Type Duplication):** Never manually define types that already exist in `shared/types/generated/`. Always import from the generated types. Duplication defeats the purpose of Tauri being the source of truth.
- **Rule 7 (IPC is the Only Data Channel):** There are no REST calls, no Next.js server actions, no API routes. All data operations go through `core/ipc/contracts/`.
- **Rule 8 (Generated Types Are Immutable):** Files inside `shared/types/generated/` must never be manually edited. They are regenerated on every Rust build.
- **Rule 9 (Controlled Public Surfaces):** Avoid blind barrel exports (`export * from './everything'`). Explicitly list public APIs to ensure clean tree-shaking and avoid circular dependencies.
- **Rule 10 (Safe Deletability):** If dropping a feature directory breaks an unrelated area, your domain boundaries are bleeding.
- **Rule 11 (Encapsulate Declarative Checks):** Do not write loose boolean conditions inside UI elements. Wrap conditions in descriptive helper functions like `canEditItem(user, item)`.
- **Rule 12 (Presentation-First UI):** UI components remain dumb and predictable. They accept parameters via props and emit events through callbacks.
- **Rule 13 (Module Gateways):** Every domain module must maintain an explicit gateway via `modules/<group>/<module>/index.ts`. External layers always go through this public contract.
- **Rule 14 (No Architectural Dumping Grounds):** Folder names like `misc/`, `helpers/`, `common/`, or `other/` are strictly banned.
- **Rule 15 (Lightweight Layout Shells):** Layout structures function purely as application shells, never as processing yards for data operations.
- **Rule 16 (Business-Blind Infrastructure):** `core/` and `core/ipc/` must never contain any code that understands your specific business domain.
- **Rule 17 (Distributed Localization):** Keep copy files adjacent to the logic that uses them: `feature/i18n/` for feature-specific strings, `module/i18n/` for module-shared strings, `shared/i18n/` for generic UI words.

---

## Pragmatic State Management

| State Type                | Recommended Tool         |
| ------------------------- | ------------------------ |
| **Local UI state**        | `useState` / `useReducer` |
| **IPC result cache**      | Redux Toolkit or Zustand |
| **Global app state**      | Redux Toolkit            |
| **Complex feature state** | Zustand / Redux          |
| **Desktop window state**  | Tauri events + Redux     |

> RTK Query is removed from this architecture. All data operations use `invoke()` wrappers from `core/ipc/contracts/`. Cache and invalidation are managed manually via Redux slices.

---

# Frontend Final Architectural Performance Metrics

| Area                    | Result    | Target Metric Met                                                           |
| ----------------------- | --------- | --------------------------------------------------------------------------- |
| **Scalability**         | High      | Unlocked parallel workflows for multi-team execution.                       |
| **Maintainability**     | High      | Code changes stay contained inside localized directories.                   |
| **Team Collaboration**  | Excellent | Minimized codebase conflicts via clear feature boundaries.                  |
| **Domain Isolation**    | Strong    | Core business rules are protected from UI framework churn.                  |
| **Refactoring Safety**  | High      | Isolated scopes make code deletion safe and reliable.                       |
| **Feature Ownership**   | Clear     | Modules track perfectly to business domains and engineering teams.          |
| **Type Safety**         | Excellent | End-to-end types from Rust structs to React components — zero duplication.  |
| **Testing**             | Easier    | Pure business layers accept straightforward unit testing.                   |
| **Cognitive Load**      | Lower     | Developers reason about a single feature directory at a time.               |

---

---

# PART 2 — BACKEND ARCHITECTURE (src-tauri)

> This section governs the `src-tauri/` directory. The Rust backend is the **single source of truth** for all shared types. It exposes functionality to the frontend exclusively via **Tauri IPC commands** — there are no HTTP endpoints, no REST API, and no Next.js server involvement.

---

# src-tauri Project Structure

```txt
src-tauri/
│
├── src/
│   ├── main.rs                        # Binary entry point — keep it 3–5 lines
│   ├── lib.rs                         # App builder: sets up state, plugins, command registration
│   │
│   ├── modules/                       # Domain modules — mirror of original server/modules/
│   │   ├── mod.rs                     # pub mod declarations for all domain groups
│   │   │
│   │   │  # ─── Domain Group: Identity & Access ─────────────────────────────
│   │   ├── identity/
│   │   │   ├── mod.rs
│   │   │   ├── users/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── commands.rs        # #[tauri::command] functions — thin delegation only
│   │   │   │   ├── domain/
│   │   │   │   │   ├── mod.rs
│   │   │   │   │   ├── entities.rs    # #[derive(Serialize, Deserialize, TS)] structs
│   │   │   │   │   ├── value_objects.rs
│   │   │   │   │   ├── errors.rs      # Domain-specific error variants
│   │   │   │   │   └── rules.rs       # Pure business rule functions
│   │   │   │   ├── services/
│   │   │   │   │   ├── mod.rs
│   │   │   │   │   ├── create_user.rs
│   │   │   │   │   └── delete_user.rs
│   │   │   │   ├── repository.rs      # sqlx SQLite data access — reads + writes
│   │   │   │   └── tests/
│   │   │   │       ├── unit/
│   │   │   │       └── integration/
│   │   │   └── authorization/
│   │   │       └── ...
│   │   │
│   │   │  # ─── Domain Group: Core Domain ──────────────────────────────────
│   │   ├── core_domain/
│   │   │   ├── mod.rs
│   │   │   ├── items/
│   │   │   │   └── ...                # Same structure as users/ above
│   │   │   └── customers/
│   │   │       └── ...
│   │   │
│   │   │  # ─── Domain Group: Operations ────────────────────────────────────
│   │   ├── operations/
│   │   │   ├── mod.rs
│   │   │   └── orders/
│   │   │       └── ...
│   │   │
│   │   └── integrations/
│   │       └── ...
│   │
│   ├── core/
│   │   ├── mod.rs
│   │   ├── error.rs                   # AppError + IPC serialization
│   │   ├── response.rs                # IpcResponse<T> + PaginatedData<T>
│   │   ├── permissions.rs             # RBAC resolver and require_permission guard
│   │   └── pagination.rs              # Pagination helpers
│   │
│   └── config/
│       ├── mod.rs
│       ├── database.rs                # SqlitePool singleton + migration runner
│       └── app.rs                     # App configuration and settings
│
├── bindings/                          # Auto-generated TypeScript types (ts-rs output)
│   ├── Item.ts                        # Generated — do not edit
│   ├── CreateItemInput.ts
│   ├── IpcResponse.ts
│   ├── PaginatedData.ts
│   └── ...
│
├── migrations/
│   └── 20240101000000_init.sql
│
├── Cargo.toml
├── build.rs                           # Triggers ts-rs export during cargo build
└── tauri.conf.json
```

---

# Domain Groups

Domain groups are **conceptual and structural boundaries** inside `src-tauri/src/modules/` that cluster related modules by business area. In Rust, they are expressed as parent modules (subdirectories with `mod.rs`) using the snake_case naming convention.

**Why use domain groups:**

- Keep related bounded contexts visible in the codebase
- Separate identity, core business, and operations concerns cleanly
- Make ownership clearer for teams
- Reduce cognitive load by making bounded contexts explicit

**Standard domain groups:**

| Domain Group      | Example Modules                      | Responsibilities                                       |
| ----------------- | ------------------------------------ | ------------------------------------------------------ |
| Identity & Access | `users`, `authorization`             | Auth, users, RBAC, roles, memberships                  |
| Core Domain       | `items`, `customers`                 | Core business resources, customer records              |
| Operations        | `orders`, `scheduling`, `catalog`    | Order processing, time slots, product catalog          |
| Integrations      | `external_services`                  | Third-party system adapters                            |

---

# TypeScript Type Generation (ts-rs)

> This is the most critical workflow in the architecture. Types flow **one direction only**: Rust → TypeScript. Never define types manually on the frontend that already exist in Rust.

## How it works

1. Annotate Rust structs with `#[derive(TS)]` and `#[ts(export)]`
2. Run `cargo build` — the `build.rs` script triggers `ts-rs` to emit `.ts` files
3. Generated files land in `src-tauri/bindings/`
4. A build script (or symlink) copies/links them to `src/frontend/shared/types/generated/`
5. Frontend code imports from `@/frontend/shared/types/generated`

## Cargo.toml setup

```toml
# src-tauri/Cargo.toml

[dependencies]
tauri = { version = "2", features = [] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio-native-tls", "migrate"] }
tokio = { version = "1", features = ["full"] }
ts-rs = { version = "10", features = ["serde-compat"] }
thiserror = "1"
```

## build.rs

```rust
// src-tauri/build.rs

fn main() {
    // Required for Tauri
    tauri_build::build();

    // ts-rs: re-run whenever any Rust source file changes
    println!("cargo:rerun-if-changed=src");
}
```

> **Note:** `ts-rs` generates type files during the test phase. Run `cargo test --features ts-rs` or rely on the `#[test]` blocks generated by `#[derive(TS)]` to trigger export. For CI, add a step that runs `cargo test` and commits updated bindings.

## Struct annotation example

```rust
// src-tauri/src/modules/core_domain/items/domain/entities.rs

use serde::{Deserialize, Serialize};
use ts_rs::TS;

/// Domain entity — exported to TypeScript automatically.
/// This struct is the single source of truth for the Item type.
#[derive(Debug, Clone, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../../../bindings/")]
pub struct Item {
    pub id: String,
    pub name: String,
    pub description: Option<String>,
    pub is_active: bool,
    pub created_at: String,
}

/// Input DTO — also exported to TypeScript.
#[derive(Debug, Clone, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../../../bindings/")]
pub struct CreateItemInput {
    pub name: String,
    pub description: Option<String>,
}

/// Update DTO
#[derive(Debug, Clone, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../../../bindings/")]
pub struct UpdateItemInput {
    pub name: Option<String>,
    pub description: Option<String>,
    pub is_active: Option<bool>,
}
```

## Generated TypeScript output (do not edit)

```typescript
// src-tauri/bindings/Item.ts — AUTO-GENERATED, DO NOT EDIT

export interface Item {
  id: string;
  name: string;
  description: string | null;
  is_active: boolean;
  created_at: string;
}
```

## Frontend import

```typescript
// src/frontend/modules/(dashboard)/items/feature-1/hooks/useItems.ts

import type { Item, CreateItemInput } from '@/frontend/shared/types/generated/Item';
import type { CreateItemInput } from '@/frontend/shared/types/generated/CreateItemInput';
```

---

# Standardized IPC Response Structure

All Tauri commands must return a unified, predictable structure for both successful results and errors. This mirrors the original API response contract, adapted for IPC.

## Rust Response Types

```rust
// src-tauri/src/core/response.rs

use serde::{Deserialize, Serialize};
use ts_rs::TS;

/// Standard success wrapper — exported to TypeScript.
#[derive(Debug, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../bindings/")]
pub struct IpcResponse<T: Serialize> {
    pub success: bool,
    pub message: String,
    pub data: Option<T>,
}

/// Paginated list wrapper — exported to TypeScript.
#[derive(Debug, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../bindings/")]
pub struct PaginatedData<T: Serialize> {
    pub items: Vec<T>,
    pub total: u64,
    pub page: u64,
    pub size: u64,
    pub pages: u64,
}

impl<T: Serialize> IpcResponse<T> {
    pub fn success(data: T, message: &str) -> Self {
        Self {
            success: true,
            message: message.to_string(),
            data: Some(data),
        }
    }
}

impl IpcResponse<()> {
    pub fn empty(message: &str) -> Self {
        Self {
            success: true,
            message: message.to_string(),
            data: None,
        }
    }
}

pub fn paginated<T: Serialize>(
    items: Vec<T>,
    total: u64,
    page: u64,
    size: u64,
    message: &str,
) -> IpcResponse<PaginatedData<T>> {
    let pages = (total as f64 / size as f64).ceil() as u64;
    IpcResponse::success(
        PaginatedData { items, total, page, size, pages },
        message,
    )
}
```

## Rust Error Types

```rust
// src-tauri/src/core/error.rs

use serde::{Deserialize, Serialize};
use ts_rs::TS;
use thiserror::Error;

/// IPC error — serialized and sent to the frontend when a command fails.
/// Tauri sends this as the rejection value of the `invoke()` Promise.
#[derive(Debug, Serialize, Deserialize, TS, Error)]
#[ts(export, export_to = "../bindings/")]
pub struct AppError {
    pub code: String,
    pub message: String,
    pub field: Option<String>,
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "[{}] {}", self.code, self.message)
    }
}

impl AppError {
    pub fn not_found(message: &str) -> Self {
        Self { code: "not_found".into(), message: message.into(), field: None }
    }
    pub fn conflict(message: &str) -> Self {
        Self { code: "conflict".into(), message: message.into(), field: None }
    }
    pub fn forbidden(message: &str) -> Self {
        Self { code: "forbidden".into(), message: message.into(), field: None }
    }
    pub fn validation(field: &str, message: &str) -> Self {
        Self { code: "validation_error".into(), message: message.into(), field: Some(field.into()) }
    }
    pub fn internal(message: &str) -> Self {
        Self { code: "internal_error".into(), message: message.into(), field: None }
    }
}

// Domain-specific errors should be defined in each module's domain/errors.rs
// and converted to AppError before crossing the command boundary.
// Example conversion in modules/core_domain/items/domain/errors.rs:
//
// #[derive(Debug, Error)]
// pub enum ItemError {
//     #[error("Item not found")]
//     NotFound,
//     #[error("Item already exists")]
//     AlreadyExists,
// }
//
// impl From<ItemError> for AppError {
//     fn from(e: ItemError) -> Self {
//         match e {
//             ItemError::NotFound => AppError::not_found("Item not found"),
//             ItemError::AlreadyExists => AppError::conflict("Item already exists"),
//         }
//     }
// }
```

---

# Commands Layer — IPC Entry Points

The `commands.rs` file in each module is the **direct equivalent of `handler.ts`** in the original server architecture. It is **thin by law** — it validates inputs, calls a service or repository, wraps the result, and returns. Zero business logic lives here.

```rust
// Example only — adapt to your project
// src-tauri/src/modules/core_domain/items/commands.rs

use tauri::State;
use crate::core::{
    error::AppError,
    response::{IpcResponse, paginated},
    permissions::require_permission,
};
use super::{
    domain::entities::{Item, CreateItemInput, UpdateItemInput},
    repository::ItemRepository,
    services::{
        create_item::CreateItemService,
        update_item::UpdateItemService,
        delete_item::DeleteItemService,
    },
};
use crate::lib::AppState;

// ── GET items ─────────────────────────────────────────────────────────────────

/// List all active items.
/// Returns a paginated list.
#[tauri::command]
pub async fn list_items(
    page: Option<u64>,
    size: Option<u64>,
    state: State<'_, AppState>,
) -> Result<IpcResponse<crate::core::response::PaginatedData<Item>>, AppError> {
    let page = page.unwrap_or(1);
    let size = size.unwrap_or(50);
    let repo = ItemRepository::new(&state.db);
    let (items, total) = repo.list_paginated(page, size).await?;
    Ok(paginated(items, total, page, size, "Items retrieved successfully"))
}

// ── GET item by id ─────────────────────────────────────────────────────────

/// Retrieve a single item by ID.
/// Returns AppError::not_found if the item does not exist.
#[tauri::command]
pub async fn get_item(
    id: String,
    state: State<'_, AppState>,
) -> Result<IpcResponse<Item>, AppError> {
    let repo = ItemRepository::new(&state.db);
    let item = repo.get_by_id(&id).await?
        .ok_or_else(|| AppError::not_found("Item not found"))?;
    Ok(IpcResponse::success(item, "Item retrieved successfully"))
}

// ── POST item ─────────────────────────────────────────────────────────────────

/// Create a new item.
/// Requires the "items:create" permission.
/// Delegates creation to CreateItemService.
#[tauri::command]
pub async fn create_item(
    data: CreateItemInput,
    state: State<'_, AppState>,
) -> Result<IpcResponse<Item>, AppError> {
    require_permission(&state, "items:create").await?;
    let service = CreateItemService::new(&state.db);
    let item = service.execute(data).await?;
    Ok(IpcResponse::success(item, "Item created successfully"))
}

// ── PUT item ──────────────────────────────────────────────────────────────────

/// Update an existing item by ID.
/// Requires the "items:update" permission.
#[tauri::command]
pub async fn update_item(
    id: String,
    data: UpdateItemInput,
    state: State<'_, AppState>,
) -> Result<IpcResponse<Item>, AppError> {
    require_permission(&state, "items:update").await?;
    let service = UpdateItemService::new(&state.db);
    let item = service.execute(&id, data).await?;
    Ok(IpcResponse::success(item, "Item updated successfully"))
}

// ── DELETE item ───────────────────────────────────────────────────────────────

/// Delete an item by ID.
/// Requires the "items:delete" permission.
#[tauri::command]
pub async fn delete_item(
    id: String,
    state: State<'_, AppState>,
) -> Result<IpcResponse<()>, AppError> {
    require_permission(&state, "items:delete").await?;
    let service = DeleteItemService::new(&state.db);
    service.execute(&id).await?;
    Ok(IpcResponse::empty("Item deleted successfully"))
}
```

---

# lib.rs — App Builder and Command Registration

The `lib.rs` file is the top-level app builder. It registers all Tauri commands, sets up application state, and initializes plugins. It is the **direct equivalent** of Next.js `app/` composition — the wiring layer.

```rust
// Example only — adapt to your project
// src-tauri/src/lib.rs

use tauri::Manager;
use sqlx::SqlitePool;

pub struct AppState {
    pub db: SqlitePool,
    // Add other shared state here (e.g., in-memory caches, config)
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            // Initialize database on startup
            let db_url = "sqlite:app.db?mode=rwc";
            let pool = tauri::async_runtime::block_on(async {
                crate::config::database::initialize(db_url).await
            })?;

            app.manage(AppState { db: pool });
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            // Items
            modules::core_domain::items::commands::list_items,
            modules::core_domain::items::commands::get_item,
            modules::core_domain::items::commands::create_item,
            modules::core_domain::items::commands::update_item,
            modules::core_domain::items::commands::delete_item,
            // Orders
            modules::operations::orders::commands::list_orders,
            modules::operations::orders::commands::create_order,
            // Users
            modules::identity::users::commands::get_current_user,
            // Add all commands here...
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

# Service Layer — Write Orchestration

Services own all write use-cases. One struct per use-case, one public method: `execute()`. The struct name carries the intent.

```rust
// Example only — adapt to your project
// src-tauri/src/modules/core_domain/items/services/create_item.rs

use sqlx::SqlitePool;
use crate::core::error::AppError;
use super::super::{
    domain::entities::{Item, CreateItemInput},
    domain::errors::ItemError,
    repository::ItemRepository,
};

pub struct CreateItemService<'a> {
    db: &'a SqlitePool,
}

impl<'a> CreateItemService<'a> {
    pub fn new(db: &'a SqlitePool) -> Self {
        Self { db }
    }

    pub async fn execute(&self, data: CreateItemInput) -> Result<Item, AppError> {
        let repo = ItemRepository::new(self.db);

        // Business rule: names must be unique
        if repo.exists_by_name(&data.name).await? {
            return Err(ItemError::AlreadyExists.into());
        }

        repo.save(&data).await.map_err(|e| AppError::internal(&e.to_string()))
    }
}
```

**Multi-step atomic transaction example:**

```rust
// Example only — adapt to your project
// src-tauri/src/modules/operations/orders/services/create_order.rs

use sqlx::SqlitePool;
use crate::core::error::AppError;
use crate::modules::core_domain::items::repository::ItemRepository;
use super::super::{
    domain::entities::{Order, CreateOrderInput},
    domain::errors::OrderError,
    repository::OrderRepository,
};

pub struct CreateOrderService<'a> {
    db: &'a SqlitePool,
}

impl<'a> CreateOrderService<'a> {
    pub fn new(db: &'a SqlitePool) -> Self {
        Self { db }
    }

    pub async fn execute(&self, data: CreateOrderInput) -> Result<Order, AppError> {
        // Cross-module read: verify the item exists
        let item_repo = ItemRepository::new(self.db);
        if item_repo.get_by_id(&data.item_id).await?.is_none() {
            return Err(OrderError::ItemNotFound.into());
        }

        // Atomic transaction for multi-step write
        let mut tx = self.db.begin().await
            .map_err(|e| AppError::internal(&e.to_string()))?;

        let order = sqlx::query_as!(
            Order,
            "INSERT INTO orders (item_id, quantity, status) VALUES (?, ?, 'pending') RETURNING *",
            data.item_id,
            data.quantity,
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(|e| AppError::internal(&e.to_string()))?;

        tx.commit().await
            .map_err(|e| AppError::internal(&e.to_string()))?;

        Ok(order)
    }
}
```

---

# Repository Layer — Data Access

The repository is the **only place that touches sqlx/SQLite**. It owns all read and write queries for its module and maps between database rows and domain entities. Read methods and write methods are clearly sectioned.

```rust
// Example only — adapt to your project
// src-tauri/src/modules/core_domain/items/repository.rs

use sqlx::{SqlitePool, FromRow};
use crate::core::error::AppError;
use super::domain::entities::{Item, CreateItemInput};

pub struct ItemRepository<'a> {
    db: &'a SqlitePool,
}

impl<'a> ItemRepository<'a> {
    pub fn new(db: &'a SqlitePool) -> Self {
        Self { db }
    }

    // ── Reads ──────────────────────────────────────────────────────────────────

    pub async fn get_by_id(&self, id: &str) -> Result<Option<Item>, AppError> {
        sqlx::query_as!(
            Item,
            "SELECT id, name, description, is_active, created_at FROM items WHERE id = ?",
            id
        )
        .fetch_optional(self.db)
        .await
        .map_err(|e| AppError::internal(&e.to_string()))
    }

    pub async fn list_paginated(&self, page: u64, size: u64) -> Result<(Vec<Item>, u64), AppError> {
        let offset = (page - 1) * size;
        let items = sqlx::query_as!(
            Item,
            "SELECT id, name, description, is_active, created_at FROM items
             WHERE is_active = true ORDER BY created_at DESC LIMIT ? OFFSET ?",
            size as i64,
            offset as i64
        )
        .fetch_all(self.db)
        .await
        .map_err(|e| AppError::internal(&e.to_string()))?;

        let total: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM items WHERE is_active = true")
            .fetch_one(self.db)
            .await
            .map_err(|e| AppError::internal(&e.to_string()))?;

        Ok((items, total as u64))
    }

    pub async fn exists_by_name(&self, name: &str) -> Result<bool, AppError> {
        let count: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM items WHERE LOWER(name) = LOWER(?)",
            name
        )
        .fetch_one(self.db)
        .await
        .map_err(|e| AppError::internal(&e.to_string()))?;
        Ok(count > 0)
    }

    // ── Writes ─────────────────────────────────────────────────────────────────

    pub async fn save(&self, data: &CreateItemInput) -> Result<Item, sqlx::Error> {
        sqlx::query_as!(
            Item,
            "INSERT INTO items (id, name, description, is_active, created_at)
             VALUES (lower(hex(randomblob(16))), ?, ?, true, datetime('now'))
             RETURNING id, name, description, is_active, created_at",
            data.name,
            data.description,
        )
        .fetch_one(self.db)
        .await
    }

    pub async fn delete(&self, id: &str) -> Result<(), AppError> {
        sqlx::query!("DELETE FROM items WHERE id = ?", id)
            .execute(self.db)
            .await
            .map_err(|e| AppError::internal(&e.to_string()))?;
        Ok(())
    }
}
```

---

# Config Layer

```rust
// Example only — adapt to your project
// src-tauri/src/config/database.rs

use sqlx::{SqlitePool, sqlite::SqlitePoolOptions};

pub async fn initialize(url: &str) -> Result<SqlitePool, sqlx::Error> {
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .connect(url)
        .await?;

    // Run migrations automatically on startup
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await?;

    Ok(pool)
}
```

---

# IPC Communication Bridge (Frontend)

The IPC layer is the **direct equivalent of the original `api/` folder**. It wraps Tauri's `invoke()` with typed command functions, normalizes errors, and maps raw responses to frontend domain models.

## core/ipc/base/ — invoke() Wrapper

```typescript
// Example only — adapt to your project
// src/frontend/core/ipc/base/invoke.ts

import { invoke as tauriInvoke } from '@tauri-apps/api/core';
import type { IpcResponse } from '@/frontend/shared/types/generated/IpcResponse';
import type { AppError } from '@/frontend/shared/types/generated/AppError';

export class IpcError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly field?: string,
  ) {
    super(message);
    this.name = 'IpcError';
  }
}

/**
 * Typed wrapper around tauri's invoke().
 * Normalizes Tauri errors into IpcError instances.
 */
export async function invoke<T>(
  command: string,
  args?: Record<string, unknown>,
): Promise<IpcResponse<T>> {
  try {
    return await tauriInvoke<IpcResponse<T>>(command, args);
  } catch (raw) {
    const error = raw as AppError;
    throw new IpcError(
      error.code ?? 'unknown_error',
      error.message ?? 'An unexpected error occurred',
      error.field ?? undefined,
    );
  }
}
```

## core/ipc/contracts/ — Per-Module Command Wrappers

```typescript
// Example only — adapt to your project
// src/frontend/core/ipc/contracts/items.ts

import { invoke } from '../base/invoke';
import type { IpcResponse } from '@/frontend/shared/types/generated/IpcResponse';
import type { PaginatedData } from '@/frontend/shared/types/generated/PaginatedData';
import type { Item } from '@/frontend/shared/types/generated/Item';
import type { CreateItemInput } from '@/frontend/shared/types/generated/CreateItemInput';
import type { UpdateItemInput } from '@/frontend/shared/types/generated/UpdateItemInput';

export const itemsCommands = {
  listItems: (
    page = 1,
    size = 50,
  ): Promise<IpcResponse<PaginatedData<Item>>> =>
    invoke('list_items', { page, size }),

  getItem: (id: string): Promise<IpcResponse<Item>> =>
    invoke('get_item', { id }),

  createItem: (data: CreateItemInput): Promise<IpcResponse<Item>> =>
    invoke('create_item', { data }),

  updateItem: (id: string, data: UpdateItemInput): Promise<IpcResponse<Item>> =>
    invoke('update_item', { id, data }),

  deleteItem: (id: string): Promise<IpcResponse<null>> =>
    invoke('delete_item', { id }),
} as const;
```

## core/ipc/mappers/ — Response to Domain Model Converters

```typescript
// Example only — adapt to your project
// src/frontend/core/ipc/mappers/item.mapper.ts

import type { Item as ItemDto } from '@/frontend/shared/types/generated/Item';

// Frontend domain model (richer than the raw IPC DTO when needed)
export interface ItemViewModel extends ItemDto {
  displayName: string;
  createdAtFormatted: string;
}

export function mapItemToViewModel(dto: ItemDto): ItemViewModel {
  return {
    ...dto,
    displayName: dto.name,
    createdAtFormatted: new Date(dto.created_at).toLocaleDateString(),
  };
}
```

## Usage in a feature hook

```typescript
// Example only — adapt to your project
// src/frontend/modules/(dashboard)/items/feature-1/hooks/useItems.ts

import { useState, useEffect } from 'react';
import { itemsCommands } from '@/frontend/core/ipc/contracts/items';
import { mapItemToViewModel, type ItemViewModel } from '@/frontend/core/ipc/mappers/item.mapper';
import { IpcError } from '@/frontend/core/ipc/base/invoke';

export function useItems() {
  const [items, setItems] = useState<ItemViewModel[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadItems = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await itemsCommands.listItems();
      if (response.data) {
        setItems(response.data.items.map(mapItemToViewModel));
      }
    } catch (e) {
      if (e instanceof IpcError) {
        setError(e.message);
      }
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => { loadItems(); }, []);

  return { items, isLoading, error, refetch: loadItems };
}
```

---

# Naming Conventions

All naming across the entire codebase must be in **English**.

---

## Rust Naming Rules

| Element                    | Convention            | Example                                                         |
| -------------------------- | --------------------- | --------------------------------------------------------------- |
| Modules / files            | `snake_case`          | `create_item.rs`, `item_repository.rs`                         |
| Structs / Enums / Traits   | `PascalCase`          | `Item`, `CreateItemInput`, `ItemError`, `ItemRepository`        |
| Functions / methods        | `snake_case`          | `list_items()`, `get_by_id()`, `execute()`                      |
| Constants                  | `SCREAMING_SNAKE_CASE`| `MAX_ITEMS_PER_PAGE`                                            |
| Type parameters            | `PascalCase`          | `T`, `E`, `TData`                                               |
| Tauri commands             | `snake_case`          | `list_items` (becomes `listItems` on frontend via Tauri)        |
| Boolean fields             | `is_` / `has_` prefix | `is_active`, `has_permission`                                   |
| Timestamps                 | `_at` / `_on` suffix  | `created_at`, `updated_at`, `expires_on`                        |
| Error enums                | `PascalCase` variant  | `ItemError::NotFound`, `OrderError::AlreadyExists`              |
| Service structs            | `<Verb><Entity>Service` | `CreateItemService`, `UpdateOrderService`, `DeleteUserService` |
| Repository structs         | `<Entity>Repository`  | `ItemRepository`, `OrderRepository`                             |

---

## TypeScript Naming Rules

| Element                     | Convention              | Example                                                          |
| --------------------------- | ----------------------- | ---------------------------------------------------------------- |
| Files                       | `kebab-case`            | `create-item.service.ts`, `use-items.hook.ts`                   |
| Interfaces / Types          | `PascalCase`            | `ItemViewModel`, `CreateItemPayload`                            |
| Classes                     | `PascalCase`            | `IpcError`, `CreateItemService`                                 |
| Functions / methods         | `camelCase`             | `listItems()`, `mapItemToViewModel()`, `useItems()`             |
| Variables                   | `camelCase`             | `isLoading`, `hasPermission`, `orderId`                         |
| Constants (module-level)    | `SCREAMING_SNAKE_CASE`  | `MAX_RETRIES`, `DEFAULT_PAGE_SIZE`                              |
| React hooks                 | `use` prefix            | `useItems`, `useItemActions`, `useItemPermissions`              |
| IPC contract objects        | `<module>Commands`      | `itemsCommands`, `ordersCommands`, `usersCommands`              |
| Mapper functions            | `map<From>To<To>()`     | `mapItemToViewModel()`, `mapOrderResponse()`                    |
| Boolean variables           | `is` / `has` / `can`    | `isActive`, `hasPermission`, `canEdit`                          |
| Collections                 | plural noun             | `items`, `orders`, `users`                                      |
| ID variables                | `<entity>Id`            | `itemId`, `orderId`                                             |

---

## IPC Command Naming Rules

Tauri automatically converts `snake_case` Rust command names to `camelCase` when called from JavaScript. Use this to your advantage:

| Rust Command Name         | JavaScript Call                      |
| ------------------------- | ------------------------------------ |
| `list_items`              | `invoke('list_items', ...)`          |
| `get_item`                | `invoke('get_item', { id })`         |
| `create_item`             | `invoke('create_item', { data })`    |
| `update_item`             | `invoke('update_item', { id, data })`|
| `delete_item`             | `invoke('delete_item', { id })`      |

**Command naming pattern:** `<verb>_<entity>` for single items, `<verb>_<entity>s` for collections.

---

## SQLite Column Naming Rules

- All column names use `snake_case`.
- Foreign key columns: `<entity>_id` (e.g., `order_id`, `user_id`)
- Boolean columns: `is_<condition>` (e.g., `is_active`, `is_deleted`)
- Timestamps: `<event>_at` (e.g., `created_at`, `updated_at`, `expires_at`)

---

# RBAC System

## Architecture Decision: Embedded RBAC

For desktop applications, RBAC is simpler than multi-tenant SaaS. Roles and permissions are stored in SQLite and cached in-process.

```rust
// Example only — adapt to your project
// src-tauri/src/core/permissions.rs

use crate::lib::AppState;
use crate::core::error::AppError;

/// Check if the current user has the given permission.
/// Reads from the in-process permission cache or SQLite.
pub async fn require_permission(
    state: &AppState,
    permission: &str,
) -> Result<(), AppError> {
    let has_perm = state.check_permission(permission).await;
    if !has_perm {
        return Err(AppError::forbidden(&format!(
            "Permission required: {}",
            permission
        )));
    }
    Ok(())
}
```

## Permission String Convention

All permission strings use explicit `resource:action` format. No wildcards. No `manage` shorthand — always expand to discrete verbs.

```
# Core domain
items:create       items:update       items:delete       items:read

# Operations
orders:create      orders:update      orders:delete      orders:read

# Settings
settings:read      settings:write

# RBAC management
roles:create       roles:update       roles:delete
members:invite     members:deactivate members:read
```

---

# Background Tasks

Background tasks in a Tauri desktop app use `tokio::spawn` instead of BullMQ. Keep them thin — all logic lives in service functions.

```rust
// Example only — adapt to your project
// src-tauri/src/modules/operations/orders/tasks/process_order.rs

use tokio::time::{interval, Duration};
use crate::lib::AppState;
use super::super::services::process_order::ProcessOrderService;
use std::sync::Arc;

/// Spawn a background task that processes pending orders every 30 seconds.
pub fn spawn_order_processor(state: Arc<AppState>) {
    tokio::spawn(async move {
        let mut ticker = interval(Duration::from_secs(30));
        loop {
            ticker.tick().await;
            let service = ProcessOrderService::new(&state.db);
            if let Err(e) = service.process_pending().await {
                eprintln!("[order_processor] Error: {}", e);
            }
        }
    });
}
```

---

# Backend Layer Responsibilities

## Infrastructure & Configuration Layers

| Layer                      | Description & Responsibilities                                                | Allowed / Contains                                                                          | Forbidden                                                             |
| -------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `config/database.rs`       | SqlitePool singleton creation and migration runner.                            | Pool initialization, `sqlx::migrate!()`, singleton export.                                 | Business logic, queries, domain rules.                                |
| `config/app.rs`            | Application configuration and environment settings.                           | Config structs, env var reading, `tauri::api::path` wrappers.                              | Application logic, module imports.                                    |
| `core/`                    | Infrastructure layer providing the system's technical, agnostic foundation.   | Base error hierarchy, IPC response wrappers, RBAC resolver, pagination helpers.             | Business-specific models, feature execution, domain rules.            |
| `core/error.rs`            | Base error hierarchy that all domain errors convert to.                       | `AppError` struct, factory methods (`not_found`, `conflict`, `forbidden`, `internal`).     | Feature-specific error variants (those live in each module's `domain/errors.rs`). |
| `core/response.rs`         | Standardized IPC response types and helper functions.                         | `IpcResponse<T>`, `PaginatedData<T>`, factory functions.                                   | Business logic, domain imports.                                       |
| `core/permissions.rs`      | Single source of truth for all RBAC resolution.                               | `require_permission()` guard, `check_permission()` resolver, permission cache.             | Business logic beyond permission checking.                            |
| `lib.rs`                   | App builder and command registration. Wiring layer only.                      | `tauri::Builder`, `invoke_handler!`, `manage()` calls, plugin initialization.              | Any logic whatsoever. This file is wiring only.                       |

---

## Module-Level Layers

| Layer                 | Description & Responsibilities                                                            | Allowed / Contains                                                                                                        | Forbidden                                                                             |
| --------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `commands.rs`         | Tauri IPC commands. Reads → call repository directly. Writes → call service.              | `#[tauri::command]` functions, `require_permission()` calls, `IpcResponse::success()` / `paginated()`.                   | Business logic, domain rules, raw sqlx queries.                                       |
| `domain/entities.rs`  | Pure, framework-independent domain structs with TypeScript export annotations.            | Rust structs and enums, `#[derive(Serialize, Deserialize, TS)]`, `#[ts(export)]`.                                         | sqlx derives, Tauri imports, any I/O.                                                |
| `domain/value_objects.rs` | Immutable, self-validating types wrapping primitives.                                | `Email`, `PhoneNumber`, `Slug` and similar types with built-in validators.                                                | Mutable state, sqlx references.                                                      |
| `domain/errors.rs`    | Domain-specific error enums that convert to `AppError`.                                   | `#[derive(Error)]` enums, `impl From<DomainError> for AppError` conversions.                                             | HTTP or IPC concerns, Tauri imports.                                                 |
| `domain/rules.rs`     | Pure business rule functions and strategy abstractions.                                   | `can_cancel_order()`, `calculate_priority()`, strategy traits and implementations.                                        | I/O operations, sqlx access, service calls.                                          |
| `services/`           | Write orchestration layer. One struct per use-case, one method: `execute()`.              | Domain imports, repository calls, `sqlx` transactions via `pool.begin()`.                                                 | `commands.rs` imports, Tauri types, raw HTTP/IPC concerns.                           |
| `repository.rs`       | Concrete sqlx data-access implementation. Owns all reads and writes for this module.      | `sqlx::query_as!()` and `sqlx::query!()` macros, `_map_to_domain()` helpers, clearly sectioned reads and writes.        | Business logic, validation, service calls.                                           |
| `tests/unit/`         | Fast, isolated unit tests with no DB or I/O.                                              | Domain entity tests, business rule tests, pure function tests, service tests with mock repos.                              | Live sqlx, real file I/O.                                                            |
| `tests/integration/`  | Full-stack tests against a real SQLite test DB.                                           | Command handler tests, service-to-DB flow tests.                                                                           | External network calls, production DB.                                               |

---

# SOLID Principles — Enforced Patterns

## Single Responsibility

One service struct per use-case, one public method: `execute()`.

```rust
// Example only

// ❌ WRONG — One struct doing multiple unrelated things
struct ItemService {
    async fn create_item(&self, data: CreateItemInput) -> Result<Item, AppError> { ... }
    async fn delete_item(&self, id: &str) -> Result<(), AppError> { ... }
    async fn archive_items(&self) -> Result<u64, AppError> { ... }
}

// ✅ CORRECT — One struct per file, one use-case per file

// services/create_item.rs
pub struct CreateItemService<'a> { db: &'a SqlitePool }
impl<'a> CreateItemService<'a> {
    pub async fn execute(&self, data: CreateItemInput) -> Result<Item, AppError> { ... }
}

// services/delete_item.rs
pub struct DeleteItemService<'a> { db: &'a SqlitePool }
impl<'a> DeleteItemService<'a> {
    pub async fn execute(&self, id: &str) -> Result<(), AppError> { ... }
}
```

## Open/Closed — Strategy Pattern

Variant behaviour is added by implementing a trait, never by editing existing service code.

```rust
// Example only
// src-tauri/src/modules/operations/orders/domain/rules.rs

pub trait PricingStrategy: Send + Sync {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64;
}

pub struct StandardPricing;
impl PricingStrategy for StandardPricing {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64 {
        base_price * quantity as f64
    }
}

pub struct BulkDiscountPricing;
impl PricingStrategy for BulkDiscountPricing {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64 {
        let discount = if quantity >= 10 { 0.9 } else { 1.0 };
        base_price * quantity as f64 * discount
    }
}
```

## Dependency Rule

All source-code dependencies point inward, toward `domain/`. Nothing inside `domain/` knows about services, repositories, or the command layer.

```
commands.rs → services/    → domain/
commands.rs → repository   → domain/
tasks/      → services/    → domain/
core/       ← commands.rs, services/
```

---

# Cross-Module Communication

When a service needs data from another module, it imports that module's repository directly. No dependency injection containers — plain imports.

```rust
// Example only — ✅ CORRECT
// src-tauri/src/modules/operations/orders/services/create_order.rs

use crate::modules::core_domain::items::repository::ItemRepository; // cross-module read

pub struct CreateOrderService<'a> { db: &'a SqlitePool }
impl<'a> CreateOrderService<'a> {
    pub async fn execute(&self, data: CreateOrderInput) -> Result<Order, AppError> {
        let item = ItemRepository::new(self.db)
            .get_by_id(&data.item_id)
            .await?
            .ok_or_else(|| AppError::not_found("Item not found"))?;
        // ... proceed with order creation
    }
}

// ❌ WRONG — Using raw sqlx in a service to read another module's data
let item = sqlx::query!("SELECT * FROM items WHERE id = ?", data.item_id)
    .fetch_optional(self.db)
    .await?; // Forbidden outside items/repository.rs
```

---

# Dependency Flow Matrix

```
                  domain   services   repository   commands   tasks   core
domain              ✅        ❌           ❌           ❌        ❌      ❌
services            ✅        ❌           ✅           ❌        ❌      ✅
repository          ✅        ❌           ❌           ❌        ❌      ❌
commands            ❌        ✅           ✅           ❌        ❌      ✅
tasks               ❌        ✅           ❌           ❌        ❌      ❌
core                ❌        ❌           ❌           ❌        ❌      ✅
```

> Cross-module: `services` and `commands` may import repositories from other modules via direct imports. The `domain/` layer never imports from any other module.

---

# Automated Enforcement (ESLint — Frontend)

```javascript
// Example only — adapt to your project
// eslint.config.mjs

import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'shared',       pattern: 'src/frontend/shared/**' },
        { type: 'core',         pattern: 'src/frontend/core/**' },
        { type: 'bootstrap',    pattern: 'src/frontend/bootstrap/**' },
        { type: 'modules',      pattern: 'src/frontend/modules/**' },
        { type: 'store',        pattern: 'src/frontend/store/**' },
        { type: 'routes',       pattern: 'src/frontend/routes/**' },
        { type: 'styles',       pattern: 'src/frontend/styles/**' },
        { type: 'app',          pattern: 'src/app/**' },
      ],
    },
    rules: {
      'boundaries/element-types': ['error', {
        default: 'disallow',
        rules: [
          // shared is the base — imports nothing above itself
          { from: 'shared',    allow: [] },
          // core can use shared
          { from: 'core',      allow: ['shared'] },
          // modules can use shared, core, and store
          { from: 'modules',   allow: ['shared', 'core', 'store'] },
          // store can use shared and core
          { from: 'store',     allow: ['shared', 'core'] },
          // routes can use modules, shared, core
          { from: 'routes',    allow: ['modules', 'shared', 'core'] },
          // bootstrap can use core and shared
          { from: 'bootstrap', allow: ['core', 'shared'] },
          // app (pages) can use modules, shared, core
          { from: 'app',       allow: ['modules', 'shared', 'core', 'store'] },
        ],
      }],
    },
  },
];
```

---

# Backend Architectural Rules

- **Rule 1 (Domain Purity):** The `domain/` layer must stay 100% pure — no sqlx imports, no Tauri types, no `AppState`, no I/O of any kind. Tests in `domain/` must pass with zero external dependencies.

- **Rule 2 (Thin Commands Layer):** `commands.rs` contains zero business logic. For reads: import and call the repository. For writes: import and call the service. No orchestration beyond that single delegation.

- **Rule 3 (Services Own All Writes):** Every write operation (create, update, delete) must go through a service struct with an `execute()` method. Never write to the DB directly from `commands.rs`.

- **Rule 4 (Repositories Own sqlx Access):** Direct `sqlx::query!()` calls are strictly forbidden everywhere except `repository.rs`. Any layer that needs data goes through the repository.

- **Rule 5 (One Service, One Use-Case):** A service struct has exactly one public method (`execute()`) performing exactly one use-case. Two actions = two service structs in two files.

- **Rule 6 (No Constructor Injection):** Services and repositories take a `&SqlitePool` reference at construction time. No DI containers. No trait-object injection. Dependencies are imported directly.

- **Rule 7 (Thin Tasks):** Background task files are thin shims — they instantiate a service and call `execute()`. Maximum 15 lines per task file. All logic lives in the service.

- **Rule 8 (Error Hierarchy):** All domain errors implement `From<DomainError> for AppError`. Commands return `Result<IpcResponse<T>, AppError>`. Services return domain errors. The conversion happens at the command boundary.

- **Rule 9 (Types Are Exported From Rust):** Every struct shared with the frontend must have `#[derive(TS)]` and `#[ts(export)]`. Never define a type manually on the frontend that has a Rust equivalent.

- **Rule 10 (Generated Bindings Are Immutable):** Files in `src-tauri/bindings/` and `src/frontend/shared/types/generated/` must never be manually edited. Regenerate them via `cargo test`.

- **Rule 11 (Commands Are the Only Public Surface):** The frontend can only interact with the backend through registered Tauri commands. No filesystem hacks, no sidecar backdoors for normal data flow.

- **Rule 12 (No Cross-Module sqlx Imports):** When a service needs data from another module, it imports that module's `repository.rs`. Direct `sqlx::query!()` for another module's data is forbidden everywhere except within that module's own repository.

- **Rule 13 (Permission Resolution via Core):** All permission checks must flow through `core/permissions.rs`. No service or repository may directly query role/permission tables for authorization purposes.

- **Rule 14 (Explicit Permission Strings):** Permission strings are explicit `resource:action` tuples. No wildcards. No `manage` shorthand — expand to `create`, `update`, `delete` individually.

- **Rule 15 (No Naming Dumping Grounds):** Folder names like `misc/`, `helpers/`, `common/`, or `other/` are strictly banned. Every folder must have a clear, bounded responsibility.

- **Rule 16 (Encapsulate Business Conditionals):** Never write loose boolean conditions inside commands or services. Wrap them in descriptive functions in `domain/rules.rs` (e.g., `can_cancel_order(&order)`).

- **Rule 17 (Safe Deletability):** If dropping a module's folder breaks an unrelated module, your domain boundaries are bleeding. Each module must be independently removable.

- **Rule 18 (lib.rs Is a Wiring File Only):** `lib.rs` registers commands and manages state. It must never contain application logic, database queries, or business decisions.

---

# Anti-Patterns Reference

| If you see this                                                   | The fix                                                                                                      |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `sqlx::query!()` inside a service struct                          | Move to `repository.rs`; import the repository in the service                                                |
| Business logic in `commands.rs`                                   | Extract to a service struct; command calls `service.execute()`                                               |
| `commands.rs` writing to DB directly via `sqlx::query!()`        | Create a service struct — commands never write directly                                                       |
| Logic inside `lib.rs` beyond command registration                 | Move all logic to the appropriate layer; `lib.rs` is wiring only                                            |
| A frontend type manually defined that matches a Rust struct       | Delete the frontend type; import from `shared/types/generated/`                                              |
| Editing a file in `shared/types/generated/`                       | STOP — those files are auto-generated; change the Rust struct and regenerate                                 |
| Calling `tauriInvoke()` directly in a module hook or component    | Route through `core/ipc/contracts/<module>.ts`; never call `invoke()` outside the contracts layer           |
| A feature importing directly from another feature                 | Bubble the shared code up to `module/shared/` or `module/domain/`                                           |
| `module/domain/` importing from `core/ipc/`                       | The `domain/` layer must never know about IPC; move the import to `feature/hooks/` or `feature/application/`|
| A layout component making IPC calls                               | Layouts are shells only; move IPC calls to the relevant module's hooks                                       |
| One service file handling two unrelated use-cases                 | Split into two files, one per use-case                                                                       |
| Permission check inside a service struct                          | Checks live only in `core/permissions.rs` via `require_permission()`; call it in `commands.rs`              |
| Domain error not implementing `From<DomainError> for AppError`    | Add the `From` impl in `domain/errors.rs`; commands should never construct `AppError` from raw strings      |
| Missing `#[derive(TS)]` on a struct shared with the frontend      | Add `#[derive(Serialize, Deserialize, TS)]` and `#[ts(export)]`; regenerate bindings                       |
| Feature-level type exposed outside feature boundaries             | Elevate the type to `module/shared/types/` or `module/domain/entities.rs`                                   |
| Strategy behaviour hardcoded with `if/else` in a service          | Create a strategy trait and implementations in `domain/rules.rs`                                             |
| Background task containing business logic                         | Task files are max 15 lines; extract all logic to a service and call `execute()` from the task              |
| Importing `AppState` from inside `domain/`                        | Domain is 100% pure — never import Tauri types or infrastructure into the domain layer                       |

---

# Server-Side Final Architectural Performance Metrics

| Area                       | Result    | Target Metric Met                                                                    |
| -------------------------- | --------- | ------------------------------------------------------------------------------------ |
| **Scalability**            | High      | Unlocked parallel workflows for multi-team execution.                                |
| **Maintainability**        | High      | Code changes stay contained inside module directories.                               |
| **Team Collaboration**     | Excellent | Minimized codebase conflicts via clear domain boundaries.                            |
| **Domain Isolation**       | Strong    | Core business rules are protected from sqlx and Tauri churn.                         |
| **Refactoring Safety**     | High      | Isolated scopes make code deletion safe and reliable.                                |
| **Feature Ownership**      | Clear     | Modules track perfectly to business domains and engineering teams.                   |
| **End-to-End Type Safety** | Excellent | Rust structs → TypeScript types via ts-rs; zero manual duplication, zero drift.      |
| **Testing**                | Easier    | Pure domain layers accept unit testing without any Tauri or sqlx setup.             |
| **Cognitive Load**         | Lower     | Developers only need to reason about one module directory at a time.                 |
| **Desktop Readiness**      | Excellent | Architecture is purpose-built for Tauri desktop; no server-side concerns leak in.   |
| **Simplicity**             | Excellent | No DI containers, no injected interfaces — direct imports reduce mental overhead.    |
| **IPC Discoverability**    | Excellent | Every command is explicitly registered in `lib.rs` and wrapped in `core/ipc/contracts/`. |

---

# Quick-Start Checklist

Use this checklist when adding a new module to the project.

## Adding a New Backend Module (src-tauri)

- [ ] Create folder under the appropriate domain group: `src-tauri/src/modules/<group>/<module>/`
- [ ] Add `mod.rs`, `commands.rs`, `repository.rs`, `domain/`, `services/`
- [ ] Define domain structs in `domain/entities.rs` with `#[derive(Serialize, Deserialize, TS)]` and `#[ts(export)]`
- [ ] Define domain errors in `domain/errors.rs` with `From<Error> for AppError`
- [ ] Implement repository read and write methods in `repository.rs`
- [ ] Create one service file per write use-case in `services/`
- [ ] Implement thin Tauri commands in `commands.rs` — one command per CRUD operation
- [ ] Register all commands in `lib.rs` inside `tauri::generate_handler![]`
- [ ] Add `pub mod <module>` in the parent `mod.rs`
- [ ] Run `cargo test` to generate updated TypeScript bindings in `src-tauri/bindings/`
- [ ] Copy or sync bindings to `src/frontend/shared/types/generated/`
- [ ] Add migrations if the module needs new tables

## Adding a New Frontend Module

- [ ] Create folder under the appropriate group: `src/frontend/modules/<group>/<module>/`
- [ ] Create `index.ts` as the public API gateway — export only what is needed
- [ ] Add IPC contract functions to `src/frontend/core/ipc/contracts/<module>.ts`
- [ ] Add mapper functions to `src/frontend/core/ipc/mappers/<module>.mapper.ts`
- [ ] Import types exclusively from `@/frontend/shared/types/generated/` — never redefine them
- [ ] Create features as needed under the module folder
- [ ] Register feature routes in `src/frontend/routes/`
- [ ] Add module i18n strings in `modules/<group>/<module>/i18n/`
