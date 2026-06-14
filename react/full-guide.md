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

# Project Structure

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
│   │   ├── products/
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

# Layer Responsibilities

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

# Architectural Rules

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



# Naming Conventions & Code Style

The naming system must be consistent across the entire frontend codebase. The goal is not cosmetic uniformity; it is faster navigation, safer refactoring, and lower cognitive load across large teams.

## General Rules

- All names must be written in **English**.
- Prefer descriptive names over abbreviations.
- Use established ecosystem abbreviations only when they are widely understood: `API`, `UI`, `URL`, `ID`, `DTO`, `RTK`, `SSR`, `CSR`.
- Do not reuse the same word for different concepts.
- Do not invent vague names such as `data`, `stuff`, `temp`, `thing`, `manager`, `helper`, or `common`.
- Keep the naming style consistent inside a single artifact type. Do not mix conventions without a clear reason.

## Folder Naming Rules

- Ordinary folders use **`kebab-case`**.
  - Examples: `dashboard-layout`, `public-layout`, `patient-records`, `appointment-history`
- Route groups use parentheses only when the framework supports them.
  - Examples: `(dashboard)`, `(public)`, `(auth)`
- Feature and module folder names should describe a bounded context or business area.
  - Good: `billing`, `appointments`, `clinic-settings`
  - Bad: `misc`, `helpers`, `other`, `stuff`
- Shared technical folders should keep their role obvious.
  - Examples: `components`, `hooks`, `utils`, `types`, `constants`, `validation`, `tests`
- Every domain module must expose a public entry point through `index.ts`.

## File Naming Rules

### React Components

- Single-component files use **`PascalCase.tsx`**.
  - Examples: `DashboardHeader.tsx`, `UserCard.tsx`, `ClinicTable.tsx`
- If a folder contains multiple component files, each file name must still match the exported component name.
- Do not use generic file names for components like `index.tsx` unless the file is intentionally a thin composition entry.

### Hooks

- Custom hooks use **`use` + PascalCase**.
  - Examples: `useAuth.ts`, `useDebounce.ts`, `useClinicFilters.ts`, `useAppointmentPermissions.ts`
- The hook name must describe the behavior, not the implementation.
- Do not prefix non-hooks with `use`.

### State and Store Files

- Redux slices use **`<feature>.slice.ts`** or **`<feature>-slice.ts`**, depending on the repository standard.
- Zustand stores use **`<feature>.store.ts`**.
- Selectors use **`<feature>.selectors.ts`**.
- RTK Query API files use **`<feature>.api.ts`**.
- Async side-effect files should clearly indicate the scope of the workflow.
  - Examples: `appointments.effects.ts`, `billing.workflow.ts`

### Utilities, Services, Mappers, Validators

- Non-component TypeScript files use **`snake_case.ts`** when the file is a pure helper, mapper, service, or validator.
  - Examples: `format_date.ts`, `map_user_dto.ts`, `calculate_totals.ts`, `validate_email.ts`
- Keep the file name aligned with the primary responsibility of the file.
- Do not place multiple unrelated responsibilities in the same file.

### Route Files

- In Next.js App Router, route entry files must use the framework names exactly:
  - `page.tsx`
  - `layout.tsx`
  - `loading.tsx`
  - `error.tsx`
  - `not-found.tsx`
  - `route.ts`
- Route folder names should use **`kebab-case`** and reflect the public URL structure.
  - Examples: `appointments`, `patient-records`, `billing-invoices`
- Route groups can be used to organize areas without changing the public route path.

### Test Files

- Test files use one of these patterns:
  - `*.test.ts`
  - `*.test.tsx`
  - `*.spec.ts`
  - `*.spec.tsx`
- The subject under test should come first and the test type should come last.
  - Examples: `use_auth.test.ts`, `UserCard.test.tsx`, `billing.service.spec.ts`
- Keep test names aligned with the thing being verified.

### Style and Asset Files

- CSS Module files should match the owning component or route name.
  - Examples: `DashboardHeader.module.css`, `user-card.module.css`
- Global style files should describe their scope.
  - Examples: `globals.css`, `theme.css`, `tokens.css`
- Static asset names should be descriptive and stable.
  - Examples: `logo-dark.svg`, `empty-state-appointments.png`

## Class and Type Naming Rules

- All class names use **`PascalCase`**.
- Component classes, providers, contexts, managers, and factories must all be named with intent.
- Never use `I` prefixes for interfaces.
- Never use `T` prefixes for types unless there is a strong team-wide convention already in place.

### Component and Provider Names

- React components use **`PascalCase`**.
  - Examples: `UserCard`, `AppointmentTable`, `ClinicSidebar`
- Context objects use **`<Domain>Context`**.
  - Example: `AuthContext`
- Context providers use **`<Domain>Provider`**.
  - Example: `AuthProvider`
- Error boundaries use **`<Domain>ErrorBoundary`**.
  - Example: `DashboardErrorBoundary`

### Domain and View Model Names

- Domain entities use **`<Concept>`**.
  - Examples: `User`, `Appointment`, `Invoice`
- View models use **`<Concept>ViewModel`** when a transformation layer is needed.
  - Examples: `UserViewModel`, `AppointmentViewModel`
- DTOs use **`<Concept>Dto`**.
  - Examples: `UserDto`, `CreateAppointmentDto`
- Request payload types use **`<Action><Concept>Input`** or **`<Action><Concept>Request`**.
  - Examples: `CreateUserInput`, `UpdateClinicRequest`
- Response types use **`<Concept>Response`** or **`<Concept>Dto`** when the transport layer is explicit.
  - Examples: `UserResponse`, `InvoiceDto`

### State and API Names

- Redux slices use **`<feature>Slice`** for the exported slice object and **`<feature>Reducer`** for the reducer output when needed.
  - Examples: `authSlice`, `appointmentsSlice`
- RTK Query APIs use **`<feature>Api`**.
  - Examples: `authApi`, `clinicApi`
- Selectors use **`select<Thing>`**.
  - Examples: `selectCurrentUser`, `selectActiveClinic`
- Action creators and handlers must describe intent.
  - Examples: `setActiveClinic`, `resetFilters`, `toggleSidebar`

### Utility and Service Class Names

- Utility classes or service objects use **`<Concept>Service`** when they perform an action-oriented workflow.
  - Examples: `NotificationService`, `AuthService`
- Pure calculation or transformation helpers should be named after the result they produce.
  - Examples: `CurrencyFormatter`, `DateRangeBuilder`, `PermissionResolver`
- Avoid naming a class `Manager` unless it truly coordinates multiple subsystems.

### Enum and Constant Names

- Enum names use **`PascalCase`**.
  - Examples: `UserRole`, `AppointmentStatus`
- Enum values use **`UPPER_SNAKE_CASE`**.
  - Examples: `ADMIN`, `PENDING`, `CANCELLED`
- Constants use **`UPPER_SNAKE_CASE`**.
  - Examples: `MAX_UPLOAD_SIZE`, `DEFAULT_PAGE_SIZE`, `API_TIMEOUT_MS`
- Boolean constants should be explicit and readable.
  - Examples: `IS_PRODUCTION`, `HAS_LIVE_DATA`

## Method and Function Naming Rules

- All function and method names use **`camelCase`** in TypeScript.
- Names must describe behavior clearly and be stable over time.

### Event Handlers

- Event handlers use **`handle` + noun/verb phrase**.
  - Examples: `handleSubmit`, `handleChange`, `handleCloseDialog`
- Props callbacks use **`on` + noun/verb phrase**.
  - Examples: `onSubmit`, `onClose`, `onSelect`
- Avoid ambiguous names like `doStuff`, `run`, or `action`.

### Data Access and Workflow Functions

- Query functions use **`get`**, **`list`**, **`find`**, or **`fetch`** depending on behavior.
  - Examples: `getUserById`, `listAppointments`, `findActiveClinic`, `fetchPatientSummary`
- Mutation functions use **`create`**, **`update`**, **`delete`**, **`archive`**, **`assign`**, or **`activate`**.
  - Examples: `createInvoice`, `updateProfile`, `deleteNotification`, `assignDoctor`
- Synchronous transformation functions use **`map`**, **`build`**, **`format`**, **`normalize`**, **`parse`**, or **`calculate`**.
  - Examples: `mapUserDtoToUser`, `buildQueryParams`, `formatCurrency`, `normalizeAppointmentStatus`
- Boolean checks use **`is`**, **`has`**, **`can`**, **`should`**, or **`supports`**.
  - Examples: `isActive`, `hasPermission`, `canEditClinic`, `shouldRetryRequest`

### React and UI-Specific Functions

- Component helper functions should be named by intent, not implementation detail.
- Derived UI selectors should be named like selectors, not generic utilities.
  - Example: `selectVisibleAppointments`
- Do not create unnamed inline business rules inside JSX when a named function would make the condition understandable.

### Async Naming

- Promise-returning functions should still describe the real action.
  - Examples: `fetchDashboardData`, `saveClinicSettings`, `loadUserPermissions`
- Do not mark names with `Async` unless the codebase has a very specific reason to do so. The action name is usually enough.

## Variable Naming Rules

- All variable names use **`camelCase`**.
- Choose names based on meaning and lifecycle, not type.
- Avoid generic names such as `data`, `item`, `value`, `tmp`, `res`, `obj`, or `stuff` unless the scope is trivial.

### Booleans

- Boolean variables use **`is`**, **`has`**, **`can`**, **`should`**, or **`needs`**.
  - Examples: `isLoading`, `hasError`, `canSubmit`, `shouldRefetch`, `needsSync`

### Collections

- Arrays and grouped collections use plural nouns.
  - Examples: `users`, `appointments`, `notifications`, `availableDoctors`

### Identifiers

- IDs use **`<entity>Id`**.
  - Examples: `userId`, `clinicId`, `appointmentId`
- Foreign identifiers follow the same rule.
  - Examples: `assignedDoctorId`, `organizationId`

### Dates and Times

- Use clear time-based names.
  - Examples: `createdAt`, `updatedAt`, `expiresAt`, `scheduledFor`

### Derived Values

- Computed values should describe the result.
  - Examples: `totalPrice`, `activeTab`, `visibleRows`, `remainingSlots`

## UI and JSX Naming Rules

- Components should be named as visual nouns or purpose-driven blocks.
  - Examples: `AppointmentCard`, `BillingSummary`, `ClinicToolbar`
- Sections inside a page should describe the part of the interface they represent.
  - Examples: `FiltersSection`, `OverviewPanel`, `ActivityFeed`
- Avoid names that encode layout mechanics instead of meaning.
  - Bad: `LeftWrapper`, `BlueBox`, `Div1`
- Prefer composition over deeply nested anonymous wrappers.

## Context, Hook, and Store Naming Rules

- Custom hooks should expose one clear responsibility.
- Hook names must start with `use`.
- Store hooks and context hooks should make the domain obvious.
  - Examples: `useAuthStore`, `useAppointmentFilters`, `useClinicPermissions`
- Context provider files should align with the exported provider name.
- Keep state-related names stable and easy to search.

## API, DTO, and Contract Naming Rules

- API files, request objects, and response objects must be named for the business action they represent.
- Backend DTOs must never leak directly into UI state or component props.
- Use mapper functions to convert transport shapes into UI-safe contracts.
- Keep contract names explicit:
  - `CreateClinicRequest`
  - `ClinicResponse`
  - `UpdateAppointmentInput`
  - `NormalizedPatientRecord`

## i18n Key Naming Rules

- Translation keys use **lowercase dot notation**.
- Keep keys stable and descriptive.
- Prefer domain-scoped keys over globally shared ambiguous keys.
  - Examples: `appointments.title`, `appointments.actions.create`, `billing.errors.payment_failed`
- Avoid free-form keys that duplicate full sentences everywhere.
- Shared generic labels should remain in the shared i18n layer.

## CSS and Styling Naming Rules

- Use class names that describe the role of the element, not the visual color or position.
  - Good: `card`, `header`, `filters`, `summary`
  - Bad: `redBox`, `topLeft`, `bigButton`
- CSS module class names should remain local to the component.
- Do not encode responsive behavior in class names. Let the CSS or utility system handle that.

## Test Naming Rules

- Test names must describe behavior.
- Prefer explicit assertion-focused names.
  - Examples: `renders user details`, `prevents submission when invalid`, `loads appointments on mount`
- Keep test file names aligned with the unit under test.
- Use the same public name as the component or function being tested.

## Boundary Naming Rules

- Public entry files should use **`index.ts`**.
- Public exports must be intentional, not blind.
- A folder is not a boundary unless it has a clear public surface.
- Names used across module boundaries should not expose internal implementation details.
- Prefer `ProfileCard` over `UserSummaryV2WidgetFactory` and similar long-term liabilities.

## Naming Examples by Layer

### shared/

- UI components: `Button`, `Dialog`, `Tabs`
- Hooks: `useDebounce`, `useOutsideClick`
- Utilities: `formatDate`, `normalizeText`
- Types: `ButtonVariant`, `NavigationItem`

### modules/

- Domain modules: `billing`, `appointments`, `notifications`
- Feature folders: `create-invoice`, `assign-doctor`, `cancel-appointment`
- Domain services: `calculateInvoiceTotal`, `canCancelAppointment`

### feature/

- Feature components: `AppointmentForm`, `PatientSummary`
- Feature hooks: `useAppointmentForm`, `usePatientSearch`
- Feature state: `appointmentSlice`, `patientSelectors`

## Naming Anti-Patterns to Avoid

- `data`, `stuff`, `thing`, `temp`, `new`, `old`
- `helper`, `helpers`, `common`, `utils` as dumping grounds
- `Manager` when the object only wraps one operation
- `Main`, `Default`, `Base` when the real meaning is available
- `Component1`, `index1`, `final`, `final_v2`, `new_final`
- Mixing casing styles in the same folder
- Copying backend naming blindly into the UI without a frontend contract layer

## Naming Priority Order

When naming is unclear, use this order:

1. Business meaning
2. Layer responsibility
3. Public API clarity
4. Technical convention
5. Conciseness

If a shorter name becomes ambiguous, the shorter name is wrong.


---

# Final Architectural Performance Metrics

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
