# Frontend Architecture (Scalable Enterprise Version)

This architecture is designed for:

* Large SaaS systems
* Multi-team development
* React / Next.js applications
* Redux Toolkit + RTK Query
* Long-term maintainability
* Feature isolation
* Domain-driven frontend systems

The goal is not just folder organization.

The real goals are:

* clear ownership
* scalability
* strict boundaries
* low coupling
* easier refactoring
* safe deletion
* team scalability
* future microfrontend readiness

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
│   ├── base/
│   ├── generated/
│   ├── contracts/
│   ├── mappers/
│   └── index.ts
│
├── layouts/
│   ├── dashboard-layout/
│   ├── auth-layout/
│   ├── public-layout/
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
│   └── assets/
│
├── modules/
│
│   ├── module-1/
│   │
│   │   ├── pages/
│   │   │
│   │   ├── domain/
│   │   │
│   │   ├── shared/
│   │   │
│   │   ├── feature-1/
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
│   │   │
│   │   └── index.ts
│   │
│   ├── module-2/
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

---

# 1. app/

Application composition layer.

Contains:

```txt
App.tsx
Providers
Theme
Router mounting
Error boundaries
Global application setup
```

## Allowed

* providers
* app composition
* app initialization
* theme setup
* router integration

## Forbidden

* business logic
* feature workflows
* domain rules
* feature state

---

# 2. bootstrap/

Application startup logic.

Contains:

```txt
environment setup
feature flags
startup configuration
service initialization
polyfills
```

---

# 3. core/

Infrastructure layer.

This is the technical foundation of the application.

Contains:

```txt
http clients
authentication infrastructure
permissions engine
logging
monitoring
analytics
error handling
configuration
storage abstraction
```

## Example

```txt
core/
  http/
  auth/
  permissions/
  monitoring/
  logging/
  config/
```

## Forbidden

* UI
* business-specific logic
* feature logic

---

# 4. api/

Backend communication layer.

This layer is separated from business modules intentionally.

---

# api/base/

Low-level API infrastructure.

Contains:

```txt
axios instances
fetch wrappers
interceptors
retry strategies
refresh token logic
timeout handling
request cancellation
base query configuration
error normalization
```

## Forbidden

* business rules
* feature-specific logic

---

# api/generated/

Auto-generated backend contracts.

Contains:

```txt
swagger generated types
openapi generated clients
DTOs
generated API contracts
```

## Critical Rule

This folder is:

* immutable
* auto-generated only

Never manually edit generated files.

---

# api/contracts/

Frontend-facing API abstractions.

Purpose:

Prevent direct dependency on backend DTOs.

Contains:

```txt
frontend api contracts
normalized response models
request abstractions
contract wrappers
```

---

# api/mappers/

Transformation layer.

Converts:

```txt
DTO -> Domain Model
Domain Model -> Request DTO
```

## Example

```ts
mapUserDtoToUser()
mapAppointmentResponse()
mapCreateRequest()
```

This prevents backend implementation details from leaking into the UI.

---

# 5. layouts/

Application shell layer.

Contains:

```txt
dashboard layouts
auth layouts
public layouts
navigation shells
page containers
```

## Responsibilities

* structure
* navigation
* layout composition
* rendering shells

## Forbidden

* business workflows
* API calls
* heavy state management

---

# 6. shared/

Global reusable resources.

This is one of the most dangerous layers in large projects.

Strict discipline is required.

---

# Allowed

```txt
shared/
  ui/
  hooks/
  utils/
  lib/
  constants/
  types/
  icons/
  assets/
```

---

# Correct Examples

```txt
shared/ui/Button
shared/hooks/useDebounce
shared/utils/date
shared/lib/currency
```

---

# Forbidden

Anything business-aware.

---

# Wrong Examples

```txt
shared/hooks/useDoctorData
shared/utils/clinicAvailability
shared/components/UserProfileCard
```

These belong inside modules.

---

# Golden Rule

If the code understands business domains:

> it does NOT belong in shared.

---

# 7. modules/

Business domain layer.

Each module represents a bounded context.

Examples:

```txt
users
clinics
billing
appointments
notifications
```

Each module should remain as isolated as possible.

---

# module/pages/

Page composition only.

## Allowed

```tsx
<PageLayout>
  <FeatureSection />
</PageLayout>
```

## Forbidden

* API calls
* business workflows
* complex orchestration

---

# module/domain/

Pure business logic layer.

Contains:

```txt
entities
business rules
domain services
pure transformations
business validations
```

## Example

```ts
canBookAppointment()
isDoctorAvailable()
calculateInvoice()
```

## Must Be

* pure
* framework independent
* testable

## Forbidden

* React
* Redux
* API calls
* browser APIs

---

# module/shared/

Reusable resources inside the module only.

Shared across features of the same module.

Contains:

```txt
shared components
shared hooks
shared utilities
shared constants
```

---

# feature-x/

Represents a use case or capability.

Not an entity.

Examples:

```txt
create-item
manage-items
bulk-update
analytics-dashboard
```

---

# feature/components/

Small UI pieces.

## Must Be

* reusable inside the feature
* mostly presentation-focused

## Forbidden

* orchestration logic
* API communication

---

# feature/sections/

Large UI compositions.

Examples:

```txt
FormSection
TableSection
OverviewSection
```

Responsibilities:

* UI composition
* feature assembly
* lightweight orchestration

---

# feature/application/

Feature workflows and orchestration.

This is one of the most important layers.

Contains:

```txt
business workflows
submit flows
modal orchestration
permission flows
optimistic updates
feature coordination
```

This layer may:

* communicate with APIs
* coordinate state
* orchestrate business actions

## Forbidden

* primitive UI rendering

---

# feature/state/

Feature state management.

Contains:

```txt
redux slices
zustand stores
selectors
actions
derived state
```

## Critical Rule

Do NOT duplicate RTK Query server state inside Redux.

---

# feature/hooks/

Feature-specific hooks.

Examples:

```txt
useFeaturePermissions
useFeatureActions
useFeatureFilters
```

## Forbidden

Globally reusable hooks.

Those belong in:

```txt
shared/hooks
```

---

# feature/utils/

Pure utilities only.

## Must Be

* stateless
* deterministic
* side-effect free

---

# feature/validation/

Validation schemas.

Examples:

```txt
zod
yup
form validation
request validation
```

---

# feature/types/

Feature-local types only.

## Rule

If the type is reused outside the feature:

> move it to module level.

---

# feature/constants/

Feature-specific constants.

---

# feature/tests/

Feature tests.

Contains:

```txt
unit tests
integration tests
feature tests
```

---

# feature/index.ts

Public API of the feature.

External layers should import only from here.

---

# Architectural Rules

---

# Rule 1

No reverse imports.

Bad:

```txt
feature-1 -> feature-2
feature-2 -> feature-1
```

---

# Rule 2

Features should not directly depend on each other internally.

Use:

* module shared
* domain layer
* orchestration
* events

---

# Rule 3

shared must never depend on modules.

Never.

---

# Rule 4

Domain layer must remain pure.

---

# Rule 5

Pages must not contain business logic.

---

# Rule 6

Do not put everything in Redux.

Use the correct tool for the correct state type.

| State Type            | Recommended Tool |
| --------------------- | ---------------- |
| Local UI state        | useState         |
| Server cache          | RTK Query        |
| Global app state      | Redux            |
| Complex feature state | Zustand / Redux  |

---

# Rule 7

RTK Query is the source of truth for server state.

Do not duplicate server data inside Redux.

---

# Rule 8

Avoid uncontrolled barrel exports.

Bad:

```ts
export * from './everything'
```

This causes:

* circular dependencies
* hidden imports
* poor tree shaking

---

# Rule 9

Every feature should be deletable safely.

If removing one feature breaks half the system:

> architecture boundaries are failing.

---

# Rule 10

Do not place business logic inside JSX.

Bad:

```tsx
user.role === 'ADMIN' && clinic.status === 'ACTIVE'
```

Better:

```ts
canManageClinic(user, clinic)
```

---

# Rule 11

Components should not become overly intelligent.

Good components:

* receive props
* render UI
* remain mostly presentation-focused

---

# Rule 12

Every module must expose a public API.

Example:

```txt
modules/module-1/index.ts
```

External layers should never import internal files directly.

---

# Rule 13

Avoid "shared dumping".

Forbidden folders:

```txt
misc/
helpers/
common/
other/
```

These become architectural garbage bins.

---

# Rule 14

Generated Swagger/OpenAPI types must never leak directly into UI components.

Use:

* contracts
* mappers
* domain models

---

# Rule 15

Layouts are application shells, not business containers.

Keep layouts lightweight.

---

# Rule 16

Infrastructure layers must remain business-agnostic.

`core/` and `api/base/` should not understand business domains.

---

# Final Result

This architecture provides:

| Area                    | Result    |
| ----------------------- | --------- |
| Scalability             | High      |
| Maintainability         | High      |
| Team Collaboration      | Excellent |
| Domain Isolation        | Strong    |
| Refactoring Safety      | High      |
| Feature Ownership       | Clear     |
| Testing                 | Easier    |
| Cognitive Load          | Lower     |
| Long-Term Stability     | Strong    |
| Microfrontend Readiness | Excellent |

