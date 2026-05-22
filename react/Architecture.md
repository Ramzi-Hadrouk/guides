# Frontend Architecture (Scalable Enterprise Version)

This architecture is designed for:

* Large SaaS systems
* Multi-team development
* React / Next.js applications
* Redux Toolkit + RTK Query
* Long-term maintainability
* Feature isolation
* Domain-driven frontend systems

The goal is not just folder organization. The real goals are **clear ownership**, **scalability**, **strict boundaries**, **low coupling**, **easier refactoring**, **safe deletion**, **team scalability**, **future microfrontend readiness**, and **localized UI ownership per module and feature**.

---

# Project Structure

```txt
src/
│
├── app/
│
├── bootstrap/
│
├── core/
│
├── api/
│   ├── base/
│   ├── generated/
│   ├── contracts/
│   ├── mappers/
│   └── index.ts
│
├── layouts/
│   ├── dashboard-layout/
│   │   ├── i18n/
│   │   ├── components/
│   │   ├── sections/
│   │   ├── application/
│   │   ├── state/
│   │   ├── hooks/
│   │   ├── utils/
│   │   ├── validation/
│   │   ├── types/
│   │   ├── constants/
│   │   ├── tests/
│   │   ├── domain/
│   │   └── index.ts  
│   ├── auth-layout/
│   ├── public-layout/
│   └── index.ts
│
├── shared/
│   ├── ui/
│   ├── hooks/
│   ├── utils/
│   ├── lib/
│   ├── constants/
│   ├── types/
│   ├── icons/
│   ├── assets/
│   └── i18n/ # Global config and utils of i18n
│
├── modules/
│   ├── module-1/
│   │   ├── i18n/
│   │   ├── pages/
│   │   ├── domain/
│   │   ├── shared/
│   │   │
│   │   ├── feature-1/
│   │   │   ├── domain/ # Feature-bound domain logic
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
│   │   │   └── index.ts
│   │   │
│   │   ├── feature-2/
│   │   └── ...
│   │   │
│   │   └── index.ts
│   │
│   ├── module-2/
│   │   └── ...
│   │
│   └── module-3/
│
├── routes/
│
├── store/
│
├── styles/
│
└── main.tsx

```

---

# Layer Responsibilities

## Root & Infrastructure Layers

| Folder / Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| app/ | Application composition layer. Handles global setup and entry wrappers. | Providers, App composition, Initialization, Theme setup, Router integration, Error boundaries. | Business logic, Feature workflows, Domain rules, Feature state. |
| bootstrap/ | Application startup logic executed before rendering the app. | Environment setup, Feature flags initialization, Startup configuration, Service initialization, Polyfills. | UI components, Application layouts, React hooks. |
| core/ | Infrastructure layer providing the system's technical, business-agnostic foundation. | HTTP clients, Auth infrastructure, Permissions engine, Logging, Monitoring, Analytics, Storage abstractions. | UI presentation code, Business-specific models, Feature execution. |
| api/base/ | Low-level network/API infrastructure. | Axios instances, Fetch wrappers, Interceptors, Retry strategies, Refresh token logic, Error normalization. | Business rules, Feature-specific logic. |
| api/generated/ | Auto-generated backend definitions. **Must remain immutable.** | Swagger/OpenAPI generated types, Generated API clients, Raw DTOs. | Manual modifications, Custom business logic mapping. |
| api/contracts/ | Frontend-facing API abstractions protecting the UI from breaking DTO modifications. | Frontend API contracts, Normalized response models, Request abstractions, Contract wrappers. | UI-specific logic, React hooks. |
| api/mappers/ | Transformation layer converting structures between client and server worlds. | mapUserDtoToUser(), mapAppointmentResponse(), DTO $\leftrightarrow$ Domain Model converters. | React primitives, Global state interactions. |
| layouts/ | Application shell layer mapping core layout layouts. | Dashboard layouts, Auth layouts, Public layouts, Navigation shells, Page containers. | Business workflows, API calls, Thick state management. |

---

## Global & Domain Shared Layers

| Folder / Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| shared/ | Purely technical global reusable items completely stripped of business awareness. | Generic components (shared/ui/Button), useDebounce, pure utilities (date, currency), global configuration (i18n/). | Domain-aware items (useDoctorData, UserProfileCard), business constraints. |
| modules/ | Business domain layer. Each subfolder represents a fully isolated bounded context. | Independent folders for domain topics (e.g., billing, appointments, notifications). | Cross-module direct file dependencies without going through public APIs. |
| module/i18n/ | Module-level localization files shared across multiple feature modules. | Shared page titles, Shared labels, Notification templates, Module-wide validation strings. | Feature-exclusive strings, Domain-agnostic generic strings. |
| module/pages/ | High-level page layout configurations and feature composition. | Layout compositions (). | Directly handling API calls, Complicated data orchestration. |
| module/domain/ | Pure, framework-independent business core logic. | Entities, Core business rules, Domain services, Pure transformations (calculateInvoice). | React components, Redux logic, API requests, Browser API dependencies. |
| module/shared/ | Reusable utilities constrained strictly within this specific business module. | Shared UI components, Hooks, and Constants unique to this module's features. | Infrastructure wrappers, Globally generic components. |

---

## Feature-Level Layers (modules/*-module/feature-x/*)

| Folder / Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| feature/domain/ | Feature-local domain and business rules required strictly to satisfy this isolated use case. | Feature specific rules, validations, and domain-scoped calculations. | React context, Hook hooks, Global actions. |
| feature/i18n/ | Micro-localization files exclusive to this particular use case. | Feature-specific button titles, Alert messages, Inline helper micro-copy. | Shared global/module strings. |
| feature/components/ | Granular UI building blocks isolated inside the local feature space. | Small presentational structures, Form inputs, Simple layout parts. | Network logic, Global state synchronization. |
| feature/sections/ | Large UI structures orchestrating presentational elements. | FormSection, TableSection, Layout assembly components. | Complex domain logic, Root framework integrations. |
| feature/application/ | Workflow coordinator and business orchestration engine. | Business workflows, Form submit orchestration, Modal toggles, Optimistic state updates. | Low-level HTML structure/Primitive rendering. |
| feature/state/ | Feature-specific client-side data state managers. | Redux slices, Zustand stores, Selectors, Local derived metrics. | Duplicating server state cache from RTK Query. |
| feature/hooks/ | Hooks scoping behaviors down to this single unique application scope. | useFeaturePermissions, useFeatureFilters, useFeatureActions. | Globally generic utility functions. |
| feature/utils/ | Functional stateless helpers serving local components. | Deterministic, side-effect-free data manipulation helpers. | Managing state configurations, Browser mutations. |
| feature/validation/ | Contract constraints evaluating runtime accuracy before processing actions. | Zod configurations, Yup schemas, Form checking workflows. | Shared cross-domain schemas. |
| feature/types/ | Data definitions defining shapes internal to this layout piece. | Component props types, Local state payload structures. | Sharing out of feature boundaries without elevating definition up to module scope. |
| feature/constants/ | Hardcoded parameters defining behavioral elements inside this component. | Action configurations, Static lists, Grid layout configurations. | Global system parameters. |
| feature/tests/ | Continuous verification suite ensuring this unit runs correctly. | Unit tests, Component integration workflows, Performance tracking files. | Cross-module logic testing configurations. |
| feature/index.ts | Public API boundary exposing designated functional capabilities outward. | Explicit item exports defining what other local folders can consume. | Exposing deep implementation details (export * from './everything'). |

---

# Architectural Rules

* **Rule 1 (No Reverse Imports):** Cross-feature direct imports are strictly forbidden. feature-1 must never import directly from feature-2.
* **Rule 2 (Feature Isolation):** Features should remain fully autonomous. Share code between features only by bubbling it up to module/shared/, the module/domain/ layer, or orchestration patterns.
* **Rule 3 (Strict Downward Flow):** shared/ must remain completely business-agnostic and never import from modules/ or core/.
* **Rule 4 (Domain Purity):** The domain/ layer must stay 100% pure: no React components, no hooks, no state managers, and no direct API calls.
* **Rule 5 (Thin View Layers):** Pages and components must keep business logic out of their views. JSX files should focus strictly on presentational mapping.
* **Rule 6 (Pragmatic State Management):** Choose state placement deliberately based on data lifecycle rules:

| State Type | Recommended Tool |
| --- | --- |
| **Local UI state** | useState / useReducer |
| **Server cache** | RTK Query |
| **Global app state** | Redux Toolkit |
| **Complex feature state** | Zustand / Redux |

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

# Final Architectural Performance Metrics

| Area | Result | Target Metric Met |
| --- | --- | --- |
| **Scalability** | High | Unlocked parallel workflows for multi-team execution. |
| **Maintainability** | High | Code changes stay contained inside localized directories. |
| **Team Collaboration** | Excellent | Minimized codebase conflicts via clear feature boundaries. |
| **Domain Isolation** | Strong | Core business rules are protected from UI framework churn. |
| **Refactoring Safety** | High | Isolated scopes make code deletion safe and reliable. |
| **Feature Ownership** | Clear | Modules track perfectly to business domains and engineering teams. |
| **Testing** | Easier | Pure business layers accept straightforward unit testing. |
| **Cognitive Load** | Lower | Developers only need to reason about a single feature directory at a time. |
| **Microfrontend Readiness** | Excellent | Features are prepared for future decomposition into standalone micro-apps. |
| **Localization Ownership** | Clear | Translation updates stay coupled with their respective features. |
