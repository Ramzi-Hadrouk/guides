# Guide 1: Core Architecture & Structure

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions. Nothing here is prescriptive for a particular industry or application.

This architecture is designed for:
- Large SaaS systems
- Multi-team development
- Django + Django-Ninja applications
- Any SQL database (PostgreSQL, MySQL, SQLite, …) + Redis + Celery + Django Channels
- Long-term maintainability
- Feature isolation
- Domain-driven backend systems

The goal is not just folder organization. The real goals are **clear ownership**, **scalability**, **strict boundaries**, **low coupling**, **easier refactoring**, **safe deletion**, **team scalability**, **future microservice readiness**, and **localized domain ownership per module**.

This architecture eliminates the selector layer and uses direct Python imports throughout — no dependency injection containers, no constructor-injected interfaces. The result is a flatter, more readable, and more maintainable codebase without sacrificing domain purity or layer boundaries.

## Project Structure

```txt
backend/
│
├── apps/
│   │
│   │   # ─── Domain Group: Identity & Access ──────────────────────────────
│   ├── users/                      # Module: users & auth
│   ├── authorization/              # Module: RBAC (roles, memberships, permissions)
│   │
│   │   # ─── Domain Group: Core Domain ────────────────────────────────────
│   ├── organizations/              # Module: tenant/organization management
│   ├── items/                      # Module: core business resource
│   ├── customers/                  # Module: customer records
│   │
│   │   # ─── Domain Group: Operations ─────────────────────────────────────
│   ├── orders/                     # Module: order processing
│   ├── scheduling/                 # Module: time slots & availability
│   ├── catalog/                    # Module: product/service catalog
│   │
│   │   # ─── Domain Group: Discovery ──────────────────────────────────────
│   ├── search/                     # Module: full-text search (read-only)
│   │
│   │   # ─── Domain Group: Integrations ───────────────────────────────────
│   ├── integrations/               # Module: third-party adapters
│   │
│   └── <module>/                   # Template — every module follows this layout
│       ├── api/
│       │   ├── router.py           # HTTP endpoints — direct imports — zero logic
│       │   └── schemas.py          # Pydantic in/out shapes only
│       ├── domain/
│       │   ├── entities.py         # Pydantic domain objects (no ORM)
│       │   ├── value_objects.py    # Immutable value types
│       │   ├── exceptions.py       # Domain-specific exceptions
│       │   └── rules.py            # Pure business rule functions / strategies
│       ├── services/               # Write use-cases (POST/PUT/PATCH/DELETE)
│       │   ├── __init__.py         # Re-exports all service classes
│       │   └── <entity>_services/  # One folder per domain entity
│       │       ├── create_<entity>_service.py
│       │       ├── update_<entity>_service.py
│       │       └── delete_<entity>_service.py
│       ├── repositories/
│       │   └── django_repo.py      # Single concrete ORM implementation
│       ├── cache/                  # Only for modules that need real-time Redis state
│       │   └── redis_repo.py
│       ├── tasks/                  # Thin Celery wrappers (one file per task)
│       ├── consumers.py            # Django Channels WebSocket consumers (if needed)
│       ├── models.py               # ORM fields + Meta. Nothing else.
│       └── tests/
│           ├── unit/               # Domain + service unit tests (no DB)
│           ├── integration/        # Router + DB integration tests
│           └── api/                # API endpoint tests (HTTP client)
│
├── core/
│   ├── exceptions.py               # Base ApplicationError + error-to-HTTP mapping
│   ├── responses.py                # Standardized API response schemas (Success, Error, Paginated)
│   ├── permissions.py              # RBAC resolver and @requires_permission decorator
│   └── pagination.py               # Shared pagination helpers
│
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   ├── production.py
│   │   └── test.py
│   ├── api.py                      # Django-Ninja root API + router registration
│   ├── celery.py                   # Celery app instance
│   ├── urls.py
│   └── asgi.py
│
├── .env
├── .env.example
└── pyproject.toml
```

> **Environment Files**
> `.env` holds actual configuration values. `.env.example` must be kept in sync whenever keys are added or removed, but only placeholder (non-sensitive) values should be committed.

## Domain Groups

Domain groups are **conceptual boundaries** inside the flat `apps/` directory that cluster related modules by business area. Because Python package names must be valid identifiers, domain groups are not enforced at the file system level — they are documented in the project structure via comments and enforced through team conventions and import-linter contracts.

**Why use domain groups:**
- Keep related bounded contexts visible in the codebase
- Separate identity, core business, operations, and discovery concerns clearly
- Make ownership clearer for teams
- Improve maintainability as the project grows
- Reduce cognitive load by making bounded contexts explicit

**Standard domain groups:**

| Domain Group | Example Modules | Responsibilities |
| --- | --- | --- |
| Identity & Access | users, authorization | Auth, users, RBAC, roles, memberships |
| Core Domain | organizations, items, customers | Tenant management, core resources, customer records |
| Operations | orders, scheduling, catalog | Order processing, time slots, product/service catalog |
| Discovery | search | Full-text search, public browsing |
| Integrations | integrations/* | Third-party system adapters |

When adding a new module, assign it to an existing domain group, or establish a new group if it represents a genuinely independent bounded context with its own entities, writes, and reads.

## Layer Responsibilities

### Infrastructure & Configuration Layers

| Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| config/settings/ | Environment-specific Django configuration. | Installed apps, database config, middleware, Celery broker, Redis URL, CORS, JWT settings. | Business logic, feature flags, domain rules. |
| config/api.py | Django-Ninja root API instance and global router registration. | `NinjaAPI` instance, global exception handlers, `api.add_router()` calls, OpenAPI configuration. | Business logic, direct endpoint definitions. |
| config/celery.py | Celery application instance and task auto-discovery. | `Celery()` instance, `autodiscover_tasks()`. | Task logic, service calls. |
| core/ | Infrastructure layer providing the system's technical, business-agnostic foundation. | Base exception hierarchy, response wrappers, RBAC resolver, permission decorator, pagination helpers. | Business-specific models, feature execution, domain rules. |
| core/exceptions.py | Base error hierarchy that all domain exceptions extend. | `ApplicationError`, `NotFoundError`, `ConflictError`, error-to-HTTP status mapping. | Feature-specific exception subclasses (those live in each module's `domain/exceptions.py`). |
| core/responses.py | Standardized API response schemas. | `ApiResponse`, `PaginatedData`, `ErrorResponse`, `ErrorDetail`. | Business logic, domain imports. |
| core/permissions.py | Single source of truth for all RBAC resolution. | `requires_permission()` decorator, `has_permission()` resolver, live DB permission lookup. | Business logic beyond permission checking, ORM queries for non-auth purposes. |

### Module-Level Layers

| Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| api/router.py | HTTP controller. For reads, imports and calls the repository directly. For writes, imports and calls the service. | `@router` endpoints, schema validation, permission decorators, direct imports of repository and service classes, OpenAPI docstrings and tags, wrapping returns in `ApiResponse`. | Business logic, domain rules, ORM model imports, transaction management outside of service calls. |
| api/schemas.py | Pydantic input/output shapes for the HTTP boundary. | Request schemas, response schemas, field-level syntax validators. | Domain entities, ORM models, business constraint validation. |
| domain/entities.py | Pure, framework-independent domain models. | Pydantic `BaseModel` domain objects, computed properties, domain-level constraints. | ORM imports, Django imports, Redis, Celery, any I/O. |
| domain/value_objects.py | Immutable, self-validating types wrapping primitives. | `Email`, `PhoneNumber`, `Slug` and similar types with built-in validators. | Mutable state, ORM references. |
| domain/exceptions.py | Domain-specific exception classes. | `OrderNotFoundError`, `SlotUnavailableError` — all subclasses of `core.exceptions.ApplicationError`. | HTTP status codes, Django exceptions. |
| domain/rules.py | Pure business rule functions and strategy abstractions. | `can_cancel_order()`, `calculate_priority()`, `PriorityStrategy` ABC and its subclasses. | I/O operations, ORM access, service calls. |
| services/ | Write orchestration layer. One class per use-case, one method: `execute()`. | Domain imports, repository imports, cache imports, `transaction.atomic()`, post-commit event broadcasting. | `api/`, HTTP objects, ORM model imports (`models.py`). |
| repositories/django_repo.py | Concrete ORM data-access implementation. Owns all reads and writes for this module. | Django ORM queries, `_map_to_domain()` mappers, `_map_to_orm()` converters. Read methods and write methods are clearly sectioned. | Business logic, validation, HTTP concerns, service calls. |
| cache/redis_repo.py | Redis interaction layer. Present only in modules that maintain live real-time state. | Redis read/write operations, key construction, TTL management, Lua scripts for atomic operations. | Services, repositories, API layer. |
| tasks/ | Thin Celery task wrappers. Max 10 lines per task file. | `@shared_task` definitions, service class import and `execute()` call. | Business logic, ORM queries, multi-step orchestration. |
| consumers.py | Django Channels WebSocket consumers. | Subscribe, receive, forward to client. Zero business logic. | Service calls, ORM queries, domain rules. |
| models.py | ORM model definition. | Field definitions, `Meta` class, `__str__`. Nothing else. | Business methods, validation logic, queryset filtering, `save()` overrides with logic. |

## SOLID Principles — Enforced Patterns

### Single Responsibility
One service class per use-case, one public method: `execute()`.

```python
# ❌ WRONG — One class doing multiple unrelated things
class OrganizationService:
    def create_organization(self, data): ...
    def update_schedule(self, data): ...
    def delete_organization(self, id): ...

# ✅ CORRECT — One class per file in a domain-specific folder

# organizations/services/organization_services/create_organization_service.py
class CreateOrganizationService:
    def execute(self, data: dict) -> Organization: ...

# organizations/services/organization_services/update_schedule_service.py
class UpdateScheduleService:
    def execute(self, organization_id: UUID, data: dict) -> Organization: ...

# organizations/services/organization_services/delete_organization_service.py
class DeleteOrganizationService:
    def execute(self, organization_id: UUID) -> None: ...
```

**Service directory example — `authorization` module:**
```txt
apps/authorization/services/
├── __init__.py
├── membership_services/
│   ├── __init__.py
│   ├── invite_member_service.py
│   ├── deactivate_member_service.py
│   └── change_role_service.py
└── role_services/
    ├── __init__.py
    ├── create_role_service.py
    └── update_permissions_service.py
```

### Open/Closed — Strategy Pattern
Variant behaviour is added by subclassing, never by editing existing service code.

```python
# apps/orders/domain/rules.py

from abc import ABC, abstractmethod


class PriorityStrategy(ABC):
    @abstractmethod
    def calculate_score(self, position: int) -> float: ...


class NormalPriority(PriorityStrategy):
    def calculate_score(self, position: int) -> float:
        return float(position)


class UrgentPriority(PriorityStrategy):
    def calculate_score(self, position: int) -> float:
        return -1000.0 + float(position)
```

Services consume the strategy via a parameter — a new priority type requires a new subclass, never an `if/elif` chain in the service.

### Liskov Substitution
Repository classes must honour the full contract they advertise. If you swap `DjangoOrderRepository` for a `MockOrderRepository` in tests, every method must behave identically in terms of return types and raised exceptions. The mock is a drop-in replacement — no special-casing allowed.

### Interface Segregation
Repositories expose only what their consumers actually call. Read-heavy flows (e.g., `search` endpoints) only touch read methods; they never import write operations. If a module's repository grows unwieldy, split it into `django_read_repo.py` and `django_write_repo.py` without changing the callers.

### Dependency Rule
All source-code dependencies point inward, toward `domain/`. Nothing inside `domain/` knows about services, repositories, or the HTTP layer. The outer rings (services, repositories, API) import domain; domain never imports outward.

```txt
api/ → services/ → domain/
api/ → repositories/ → domain/
cache/ → domain/
tasks/ → services/ → domain/
core/ ← api/, services/
```

## Cross-Module Communication

When a service or the API needs data from another module, it imports that module's `django_repo.py` directly. No injected interfaces, no adapter layers — just a plain import.

```python
# ✅ CORRECT — service reads from another module via direct repository import
# apps/orders/services/order_services/create_order_service.py

from apps.orders.repositories.django_repo import DjangoOrderRepository
from apps.items.repositories.django_repo import DjangoItemRepository
from apps.authorization.repositories.django_repo import DjangoAuthorizationRepository


class CreateOrderService:
    def execute(self, data: dict) -> Order:
        if not DjangoItemRepository().get_by_id(data["item_id"]):
            raise ItemNotFoundError()

        with transaction.atomic():
            return DjangoOrderRepository().save(data)


# ✅ CORRECT — router assembles a composite read from multiple modules
# apps/search/api/router.py

from apps.search.repositories.django_repo import DjangoSearchRepository
from apps.catalog.repositories.django_repo import DjangoCatalogRepository


@router.get("/search/", response=ApiResponse[list[SearchResultSchema]])
def search(request, q: str, organization_id: UUID | None = None):
    """
    Full-text search across the platform.

    Returns matching results from the search index.
    """
    repo = DjangoSearchRepository()
    results = repo.search(organization_id=organization_id, query=q)
    return ApiResponse(message="Search completed successfully", data=results)


# ❌ WRONG — importing another module's ORM model directly
from apps.items.models import ItemORM              # forbidden outside items/repositories/
from apps.authorization.models import RoleORM      # forbidden outside authorization/repositories/
from apps.catalog.models import CatalogItemORM     # forbidden outside catalog/repositories/
```

## Dependency Flow Matrix

```txt
                domain   services   repositories   api     tasks   cache   core
domain            ✅        ❌           ❌          ❌       ❌      ❌      ❌
services          ✅        ❌           ✅           ❌       ❌      ✅      ✅
repositories      ✅        ❌           ❌           ❌       ❌      ❌      ❌
api               ❌        ✅           ✅           ❌       ❌      ❌      ✅
tasks             ❌        ✅           ❌           ❌       ❌      ❌      ❌
cache             ✅        ❌           ❌           ❌       ❌      ❌      ❌
core              ❌        ❌           ❌           ❌       ❌      ❌      ✅
```

> Cross-module: services and API may import repositories from other modules via direct Python imports. The `domain/` layer never imports from any other module.

## Automated Enforcement (pyproject.toml)

```toml
[tool.importlinter]
root_packages = ["apps", "core"]
include_external_packages = true

[[tool.importlinter.contracts]]
name = "Domain layer is fully independent"
type = "forbidden"
source_modules = ["apps.*.domain"]
forbidden_modules = [
    "apps.*.services",
    "apps.*.repositories",
    "apps.*.cache",
    "apps.*.api",
    "django",
    "redis",
    "celery",
]

[[tool.importlinter.contracts]]
name = "Core is a shared leaf — never imports from apps"
type = "forbidden"
source_modules = ["core"]
forbidden_modules = ["apps"]

[[tool.importlinter.contracts]]
name = "Services do not touch the API or ORM models directly"
type = "forbidden"
source_modules = ["apps.*.services"]
forbidden_modules = [
    "apps.*.api",
    "apps.*.models",
]

[[tool.importlinter.contracts]]
name = "API never imports ORM or cache directly"
type = "forbidden"
source_modules = ["apps.*.api"]
forbidden_modules = [
    "apps.*.cache",
    "apps.*.models",
]

[[tool.importlinter.contracts]]
name = "Cache layer only talks to domain and infrastructure"
type = "forbidden"
source_modules = ["apps.*.cache"]
forbidden_modules = [
    "apps.*.services",
    "apps.*.api",
    "apps.*.repositories",
]

[[tool.importlinter.contracts]]
name = "Tasks only call services — no ORM or API access"
type = "forbidden"
source_modules = ["apps.*.tasks"]
forbidden_modules = [
    "apps.*.api",
    "apps.*.models",
]

[[tool.importlinter.contracts]]
name = "core/permissions.py is the only RBAC entry point"
type = "forbidden"
source_modules = ["apps.*.services", "apps.*.api"]
forbidden_modules = ["apps.authorization.repositories.django_repo"]
# All permission resolution must go through core/permissions.py

[[tool.importlinter.contracts]]
name = "search module is read-only — no services allowed"
type = "forbidden"
source_modules = ["apps.search"]
forbidden_modules = ["apps.search.services"]
```

