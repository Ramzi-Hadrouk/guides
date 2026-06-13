# Django-Ninja Architecture Guide (Scalable Enterprise Version)

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions. Nothing here is prescriptive for a particular industry or application.

---

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

---

# Project Structure

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
│           └── integration/        # Router + DB integration tests
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
>
> `.env` holds actual configuration values. `.env.example` must be kept in sync whenever keys are added or removed, but only placeholder (non-sensitive) values should be committed.

---

# Domain Groups

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

---

# Naming Conventions

All naming across the entire codebase must be in **English**. This ensures consistency, readability for international teams, and alignment with Python and Django ecosystem conventions.

---

## File Naming Rules

- All file names use **`snake_case`**.
- Service files follow the pattern: **`<verb>_<entity>_service.py`**
  - `create_order_service.py`, `update_user_service.py`, `delete_item_service.py`
- Repository files: **`django_repo.py`**, **`redis_repo.py`** (one per module, named by infrastructure)
- API files: **`router.py`**, **`schemas.py`** (always these exact names inside `api/`)
- Domain files: **`entities.py`**, **`value_objects.py`**, **`exceptions.py`**, **`rules.py`**
- Task files: **`<descriptive_action>_task.py`**
  - `send_welcome_email_task.py`, `process_order_task.py`
- Test files: **`test_<feature>_<layer>.py`**
  - `test_order_service_unit.py`, `test_item_router_integration.py`
- Consumer files: **`consumers.py`**

---

## Class Naming Rules

- All class names use **`PascalCase`**.
- ORM model classes: **`<Entity>ORM`**
  - `UserORM`, `OrderORM`, `OrganizationORM`, `ItemORM`
- Domain entity classes: **`<Entity>`**
  - `User`, `Order`, `Organization`, `Item`
- Repository classes: **`Django<Entity>Repository`** or **`Redis<Entity>Repository`**
  - `DjangoUserRepository`, `DjangoOrderRepository`, `RedisAuthorizationRepository`
- Service classes: **`<Verb><Entity>Service`**
  - `CreateOrderService`, `UpdateUserService`, `DeleteItemService`, `LoginService`
- Schema classes: **`<Verb><Entity><Direction>Schema`**
  - `CreateOrderRequestSchema`, `OrderResponseSchema`, `UpdateItemRequestSchema`
- Exception classes: **`<Entity><Condition>Error`**
  - `OrderNotFoundError`, `UserAlreadyExistsError`, `SlotUnavailableError`
- Value Object classes: **`<Concept>`**
  - `Email`, `PhoneNumber`, `Money`, `Slug`
- Strategy ABCs: **`<Concept>Strategy`**
  - `PriorityStrategy`, `PricingStrategy`, `DiscountStrategy`

---

## Method Naming Rules

- All method names use **`snake_case`**.
- Repository read methods: **`get_by_<field>`**, **`list_by_<field>`**, **`exists_by_<field>`**
  - `get_by_id()`, `list_by_organization()`, `exists_by_email()`
- Repository write methods: **`save`**, **`update`**, **`delete`**, **`bulk_create`**
- Service public method: **`execute`** — always this exact name; the class name carries the intent
- Domain rule functions: **`can_<action>`**, **`calculate_<result>`**, **`validate_<condition>`**
  - `can_cancel_order()`, `calculate_priority()`, `validate_membership()`
- Mapper methods (private): **`_map_to_domain`**, **`_map_to_orm`**

---

## Variable Naming Rules

- All variable names use **`snake_case`**.
- Boolean variables: **`is_<condition>`**, **`has_<feature>`**, **`can_<action>`**
  - `is_active`, `has_permission`, `can_cancel`
- Collections: plural form of the entity
  - `orders`, `users`, `items`
- IDs: **`<entity>_id`**
  - `order_id`, `user_id`, `organization_id`
- Timestamps: **`<event>_<time_unit>`**
  - `created_at`, `updated_at`, `expires_on`

---

## URL / Endpoint Naming Rules

- URL segments use **`kebab-case`**.
- Plural nouns for collections: **`/api/v1/orders/`**
- Nested resources: **`/api/v1/organizations/{org_id}/members/`**
- Specific actions on a resource: **`/api/v1/orders/{order_id}/cancel/`**

---

## Redis Key Naming Rules

- All Redis key segments use **English**.
- Pattern: **`<domain>:<id>:<type>`**
- Examples:
  - `queue:{queue_id}:counter` → STRING — atomic INCR
  - `queue:{queue_id}:state` → HASH — live state
  - `slots:{org_id}:{catalog_id}:{date}` → HASH — TTL 60s
  - `perm:{org_id}:{user_id}` → STRING — TTL 300s
  - `search:{org_id}:{query}:{month}` → STRING — TTL 3600s
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

- All schema field names use **`snake_case`**.
- Follow the same rules as variable naming above.
- Examples: `order_id`, `status`, `created_at`, `is_active`, `duration_min`

---

## ORM Model Field Naming Rules

- All field names use **`snake_case`**.
- Follow the same rules as variable naming above.
- Foreign key fields: **`<entity>_id`** (e.g., `organization_id`, `user_id`)
- Examples: `ticket_number`, `status`, `created_at`, `is_active`, `duration_min`

---

# Standardized API Response Structure

All API endpoints must return a unified, predictable JSON structure for both successful requests and errors. This ensures frontend clients, mobile apps, and third-party integrations can parse responses consistently without special-casing endpoints.

---

## Core Response Schemas

The standard response envelopes are defined in `core/responses.py` using Pydantic generics. This allows type-safe payloads while maintaining a uniform outer shell.

```python
# Example only — adapt to your project

# core/responses.py

from typing import Any, Generic, TypeVar, Optional
from pydantic import BaseModel


T = TypeVar("T")


class ErrorDetail(BaseModel):
    """Schema for individual validation or business rule errors."""
    field: Optional[str] = None
    message: str
    code: Optional[str] = None  # Machine-readable error code (e.g., "required", "invalid_format")


class ApiResponse(BaseModel, Generic[T]):
    """
    Standard envelope for all successful responses.
    """
    success: bool = True
    message: str = "Request successful"
    data: Optional[T] = None


class PaginatedData(BaseModel, Generic[T]):
    """
    Wrapper for paginated list data.
    """
    items: list[T]
    total: int
    page: int
    size: int
    pages: int


class ErrorResponse(BaseModel):
    """
    Standard envelope for all error responses.
    """
    success: bool = False
    message: str
    errors: Optional[list[ErrorDetail]] = None
```

---

## Success Response Contract

Every successful endpoint response must be wrapped in `ApiResponse[T]`, where `T` is the specific schema of the returned data.

**Single Item Response:**
```json
{
  "success": true,
  "message": "Item retrieved successfully",
  "data": {
    "id": "uuid",
    "name": "Example Item",
    "is_active": true
  }
}
```

**List Response (Paginated):**
```json
{
  "success": true,
  "message": "Items retrieved successfully",
  "data": {
    "items": [
      { "id": "uuid", "name": "Item 1", "is_active": true }
    ],
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

All errors—whether validation errors, business rule violations, or system errors—must return the `ErrorResponse` schema with appropriate HTTP status codes.

**Business Logic Error (e.g., 400, 403, 404, 409):**
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

**Validation Error (e.g., 422):**
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format",
      "code": "invalid_format"
    },
    {
      "field": "quantity",
      "message": "Ensure this value is greater than 0",
      "code": "min_value"
    }
  ]
}
```

---

## Global Exception Handler

To enforce the error response structure automatically, register a global exception handler in `config/api.py`. Domain exceptions are caught and mapped to the `ErrorResponse`.

```python
# Example only — adapt to your project

# config/api.py

from ninja import NinjaAPI
from ninja.errors import HttpError, ValidationError
from core.exceptions import ApplicationError
from core.responses import ErrorResponse, ErrorDetail

api = NinjaAPI(
    title="Platform API",
    version="1.0.0",
    docs_url="/docs/",
    openapi_url="/openapi.json",
)


@api.exception_handler(ApplicationError)
def application_error_handler(request, exc: ApplicationError):
    """Handle custom domain exceptions."""
    return api.create_response(
        request,
        ErrorResponse(
            message=exc.message,
            errors=[ErrorDetail(field=exc.field, message=exc.message, code=exc.code)],
        ),
        status_code=exc.status_code,
    )


@api.exception_handler(ValidationError)
def validation_error_handler(request, exc: ValidationError):
    """Handle Pydantic/Django-Ninja validation errors uniformly."""
    errors = [
        ErrorDetail(field=".".join(str(loc) for loc in err["loc"]), message=err["msg"], code=err["type"])
        for err in exc.errors
    ]
    return api.create_response(
        request,
        ErrorResponse(message="Validation failed", errors=errors),
        status_code=422,
    )


@api.exception_handler(HttpError)
def http_error_handler(request, exc: HttpError):
    """Handle standard HTTP errors (e.g., 403, 404)."""
    return api.create_response(
        request,
        ErrorResponse(message=exc.message),
        status_code=exc.status_code,
    )
```

---

## Base Domain Exception

Update `core/exceptions.py` to support the fields required by the error handler:

```python
# Example only — adapt to your project

# core/exceptions.py


class ApplicationError(Exception):
    """Base exception for all domain/business errors."""

    def __init__(
        self,
        message: str = "An application error occurred",
        status_code: int = 400,
        code: str | None = None,
        field: str | None = None,
    ):
        self.message = message
        self.status_code = status_code
        self.code = code
        self.field = field
        super().__init__(self.message)


class NotFoundError(ApplicationError):
    def __init__(self, message: str = "Resource not found", field: str | None = None):
        super().__init__(message=message, status_code=404, code="not_found", field=field)


class ConflictError(ApplicationError):
    def __init__(self, message: str = "Resource conflict", field: str | None = None):
        super().__init__(message=message, status_code=409, code="conflict", field=field)
```

---

# Layer Responsibilities

## Infrastructure & Configuration Layers

| Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| config/settings/ | Environment-specific Django configuration. | Installed apps, database config, middleware, Celery broker, Redis URL, CORS, JWT settings. | Business logic, feature flags, domain rules. |
| config/api.py | Django-Ninja root API instance and global router registration. | `NinjaAPI` instance, global exception handlers, `api.add_router()` calls, OpenAPI configuration. | Business logic, direct endpoint definitions. |
| config/celery.py | Celery application instance and task auto-discovery. | `Celery()` instance, `autodiscover_tasks()`. | Task logic, service calls. |
| core/ | Infrastructure layer providing the system's technical, business-agnostic foundation. | Base exception hierarchy, response wrappers, RBAC resolver, permission decorator, pagination helpers. | Business-specific models, feature execution, domain rules. |
| core/exceptions.py | Base error hierarchy that all domain exceptions extend. | `ApplicationError`, `NotFoundError`, `ConflictError`, error-to-HTTP status mapping. | Feature-specific exception subclasses (those live in each module's `domain/exceptions.py`). |
| core/responses.py | Standardized API response schemas. | `ApiResponse`, `PaginatedData`, `ErrorResponse`, `ErrorDetail`. | Business logic, domain imports. |
| core/permissions.py | Single source of truth for all RBAC resolution. | `requires_permission()` decorator, `has_permission()` resolver, live DB permission lookup. | Business logic beyond permission checking, ORM queries for non-auth purposes. |

---

## Module-Level Layers

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
| tests/unit/ | Fast, isolated unit tests with no DB or network I/O. | Domain entity tests, business rule tests, pure function tests, service tests with mocked repos. | Live ORM, HTTP calls, real Redis. |
| tests/integration/ | Full-stack tests against the real DB. | Router endpoint tests, service-to-DB flow tests. | Production Redis, external third-party calls. |

---

# OpenAPI Documentation

All endpoints must be documented using Django-Ninja's built-in OpenAPI support. The OpenAPI spec is the single source of truth for API consumers and must stay accurate.

---

## Root API Configuration

Configure the `NinjaAPI` instance in `config/api.py` with meaningful metadata:

```python
# Example only — adapt to your project

from ninja import NinjaAPI

api = NinjaAPI(
    title="Platform API",
    version="1.0.0",
    description="API documentation for the platform. All endpoints require authentication unless explicitly marked as public.",
    docs_url="/docs/",
    openapi_url="/openapi.json",
)
```

---

## Router Tags

Every module's router must declare a **tag** that groups its endpoints in the OpenAPI UI. Tags use the module's English name in plural form, `PascalCase` for display.

```python
# Example only — adapt to your project

from ninja import Router

router = Router(tags=["Orders"])
```

Register the router with the same tag in `config/api.py`:

```python
# Example only — adapt to your project

from config.api import api
from apps.orders.api.router import router as orders_router

api.add_router("/orders/", orders_router, tags=["Orders"])
```

---

## Endpoint Docstrings

Every endpoint function must have a **docstring** that describes what it does. The first line is the summary; subsequent lines are the description.

```python
# Example only — adapt to your project

@router.get("/orders/{order_id}/", response=ApiResponse[OrderResponseSchema])
def get_order(request, order_id: UUID):
    """
    Retrieve a single order by ID.

    Returns the full order details including items, status, and timestamps.
    Raises 404 if the order does not exist.
    """
    ...
```

---

## Response and Request Annotations

All endpoints must declare their `response` type using the `ApiResponse[T]` wrapper schema. For `POST`/`PUT`/`PATCH`, the payload schema must be explicitly typed as a parameter.

```python
# Example only — adapt to your project

@router.post("/orders/", response=ApiResponse[OrderResponseSchema])
def create_order(request, payload: CreateOrderRequestSchema):
    """Create a new order."""
    ...


@router.get("/orders/", response=ApiResponse[PaginatedData[OrderResponseSchema]])
def list_orders(request, organization_id: UUID):
    """List all orders for a given organization."""
    ...
```

---

## Operation IDs

For endpoints that need stable, client-facing operation IDs (e.g., code generation), use the `operation_id` parameter:

```python
# Example only — adapt to your project

@router.get("/orders/{order_id}/", response=ApiResponse[OrderResponseSchema], operation_id="getOrder")
def get_order(request, order_id: UUID):
    """Retrieve a single order by ID."""
    ...
```

---

## Deprecated Endpoints

When an endpoint is being phased out, mark it with `deprecated=True` instead of removing it immediately:

```python
# Example only — adapt to your project

@router.get("/orders/legacy/list/", response=ApiResponse[PaginatedData[OrderResponseSchema]], deprecated=True)
def list_orders_legacy(request, organization_id: UUID):
    """
    [DEPRECATED] Use GET /orders/ instead.

    Legacy list endpoint retained for backward compatibility.
    Will be removed in v2.
    """
    ...
```

---

## OpenAPI Rules Summary

1. **Every endpoint must have a docstring** — first line is the summary, the rest is the description.
2. **Every router must declare a tag** matching its module name.
3. **Every endpoint must declare its `response` type** via the `response` parameter, wrapped in `ApiResponse[T]` or `ApiResponse[PaginatedData[T]]`.
4. **Every `POST`/`PUT`/`PATCH` payload must be a typed schema parameter**, never raw `dict` or `Request`.
5. **Use `operation_id`** when the auto-generated ID is not stable or readable enough for client generation.
6. **Mark deprecated endpoints with `deprecated=True`**, do not silently remove them.
7. **Keep the docstring in sync with the endpoint behavior** — stale docs are worse than no docs.

---

# SOLID Principles — Enforced Patterns

## Single Responsibility

One service class per use-case, one public method: `execute()`.

```python
# Example only — adapt to your project

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

```
# Example only — adapt to your project

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

---

## Open/Closed — Strategy Pattern

Variant behaviour is added by subclassing, never by editing existing service code.

```python
# Example only — adapt to your project

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

---

## Liskov Substitution

Repository classes must honour the full contract they advertise. If you swap `DjangoOrderRepository` for a `MockOrderRepository` in tests, every method must behave identically in terms of return types and raised exceptions. The mock is a drop-in replacement — no special-casing allowed.

---

## Interface Segregation

Repositories expose only what their consumers actually call. Read-heavy flows (e.g., `search` endpoints) only touch read methods; they never import write operations. If a module's repository grows unwieldy, split it into `django_read_repo.py` and `django_write_repo.py` without changing the callers.

---

## Dependency Rule

All source-code dependencies point inward, toward `domain/`. Nothing inside `domain/` knows about services, repositories, or the HTTP layer. The outer rings (services, repositories, API) import domain; domain never imports outward.

```
api/ → services/ → domain/
api/ → repositories/ → domain/
cache/ → domain/
tasks/ → services/ → domain/
core/ ← api/, services/
```

---

# Service Layer — Write Orchestration

Services own all write use-cases. They import repositories directly via normal Python imports — no constructor injection, no interface parameters.

```python
# Example only — adapt to your project

# apps/items/services/item_services/create_item_service.py

from django.db import transaction

from apps.items.repositories.django_repo import DjangoItemRepository
from apps.items.domain.entities import Item
from apps.items.domain.exceptions import ItemAlreadyExistsError


class CreateItemService:
    def execute(self, data: dict) -> Item:
        repo = DjangoItemRepository()

        if repo.get_by_name(data["name"]):
            raise ItemAlreadyExistsError()

        with transaction.atomic():
            return repo.save(data)
```

**Real-world examples across modules:**

```python
# Example only — adapt to your project

# apps/users/services/auth_services/login_service.py
class LoginService:
    def execute(self, email: str, password: str) -> TokenPair: ...

# apps/organizations/services/organization_services/register_organization_service.py
class RegisterOrganizationService:
    """
    Organization self-registration — one atomic transaction:
    1. Create organization record (with owner_user_id)
    2. Seed organization_roles (ADMIN, MANAGER, MEMBER)
    3. Seed role_permissions per default matrix
    4. Create organization_memberships for owner → ADMIN role
    """
    def execute(self, data: dict) -> Organization: ...

# apps/authorization/services/membership_services/invite_member_service.py
class InviteMemberService:
    def execute(self, data: dict) -> dict: ...
```

**Cross-module reads inside a write service** — import the other module's repository directly:

```python
# Example only — adapt to your project

# apps/orders/services/order_services/create_order_service.py

from django.db import transaction

from apps.orders.repositories.django_repo import DjangoOrderRepository
from apps.items.repositories.django_repo import DjangoItemRepository       # cross-module read
from apps.orders.domain.exceptions import ItemNotFoundError, SlotUnavailableError


class CreateOrderService:
    def execute(self, data: dict) -> Order:
        item_repo = DjangoItemRepository()
        if not item_repo.get_by_id(data["item_id"]):
            raise ItemNotFoundError()

        repo = DjangoOrderRepository()
        with transaction.atomic():
            return repo.save(data)
```

---

# Repository Layer — Data Access

The repository is the **only place that touches the ORM**. It owns all read and write queries for its module and maps between ORM models and domain entities. Read methods and write methods are clearly sectioned inside the same class.

```python
# Example only — adapt to your project

# apps/items/repositories/django_repo.py

from uuid import UUID

from apps.items.models import ItemORM
from apps.items.domain.entities import Item


class DjangoItemRepository:

    # ── Reads ─────────────────────────────────────────────────────────────────

    def get_by_id(self, item_id: UUID) -> Item | None:
        try:
            return self._map_to_domain(ItemORM.objects.get(id=item_id))
        except ItemORM.DoesNotExist:
            return None

    def get_by_name(self, name: str) -> Item | None:
        try:
            return self._map_to_domain(ItemORM.objects.get(name=name.lower()))
        except ItemORM.DoesNotExist:
            return None

    def list_by_organization(self, organization_id: UUID) -> list[Item]:
        return [
            self._map_to_domain(orm)
            for orm in ItemORM.objects.filter(organization_id=organization_id, is_active=True)
        ]

    # ── Writes ────────────────────────────────────────────────────────────────

    def save(self, data: dict) -> Item:
        orm = ItemORM.objects.create(**self._map_to_orm(data))
        return self._map_to_domain(orm)

    def update(self, item_id: UUID, data: dict) -> Item:
        ItemORM.objects.filter(id=item_id).update(**self._map_to_orm(data))
        return self._map_to_domain(ItemORM.objects.get(id=item_id))

    def delete(self, item_id: UUID) -> None:
        ItemORM.objects.filter(id=item_id).delete()

    # ── Mappers ───────────────────────────────────────────────────────────────

    def _map_to_domain(self, orm: ItemORM) -> Item:
        return Item(
            id=orm.id,
            name=orm.name,
            description=orm.description,
            is_active=orm.is_active,
        )

    def _map_to_orm(self, data: dict) -> dict:
        return {
            "name": data["name"].lower(),
            "description": data["description"],
        }
```

---

# API Layer — HTTP Controllers

The API layer is **thin by law**. It delegates immediately — never implements. For reads (GET), it imports and calls the repository. For writes (POST/PUT/PATCH/DELETE), it imports and calls the service. All return types use the standardized `ApiResponse` wrapper.

```python
# Example only — adapt to your project

# apps/items/api/router.py

from uuid import UUID
from ninja import Router
from ninja.errors import HttpError

from core.permissions import requires_permission
from core.responses import ApiResponse, PaginatedData
from apps.items.api.schemas import CreateItemRequestSchema, ItemResponseSchema
from apps.items.repositories.django_repo import DjangoItemRepository
from apps.items.services.item_services.create_item_service import CreateItemService
from apps.items.services.item_services.delete_item_service import DeleteItemService

router = Router(tags=["Items"])


# ── GET ───────────────────────────────────────────────────────────────────────

@router.get("/organizations/{organization_id}/items/", response=ApiResponse[PaginatedData[ItemResponseSchema]])
def list_items(request, organization_id: UUID):
    """
    List all active items for an organization.

    Returns a paginated list of items filtered by the given organization.
    """
    repo = DjangoItemRepository()
    items = repo.list_by_organization(organization_id)
    # Note: Pagination logic omitted for brevity; apply core.pagination helpers here
    return ApiResponse(
        message="Items retrieved successfully",
        data=PaginatedData(items=items, total=len(items), page=1, size=50, pages=1),
    )


@router.get("/items/{item_id}/", response=ApiResponse[ItemResponseSchema])
def get_item(request, item_id: UUID):
    """
    Retrieve a single item by ID.

    Returns the full item details. Raises 404 if not found.
    """
    repo = DjangoItemRepository()
    item = repo.get_by_id(item_id)
    if not item:
        raise HttpError(404, "Item not found")
    return ApiResponse(message="Item retrieved successfully", data=item)


# ── POST / PUT / DELETE ───────────────────────────────────────────────────────

@router.post("/organizations/{organization_id}/items/", response=ApiResponse[ItemResponseSchema])
@requires_permission("items:create")
def create_item(request, organization_id: UUID, payload: CreateItemRequestSchema):
    """
    Create a new item within an organization.

    Validates the payload and delegates creation to the service layer.
    """
    service = CreateItemService()
    item = service.execute(payload.dict() | {"organization_id": organization_id})
    return ApiResponse(message="Item created successfully", data=item)


@router.delete("/items/{item_id}/", response=ApiResponse)
@requires_permission("items:delete")
def delete_item(request, item_id: UUID):
    """
    Delete an item by ID.

    Permanently removes the item. Requires items:delete permission.
    """
    service = DeleteItemService()
    service.execute(item_id)
    return ApiResponse(message="Item deleted successfully", data=None)
```

---

# RBAC System

> **This section governs `core/permissions.py` and the `authorization` module.** The permission decorator touches every API endpoint in every module, so the contract here is non-negotiable.

## Architecture Decision: Manual Tenant-Scoped RBAC

The platform uses manual RBAC instead of a third-party library (django-guardian, django-rules, django-role-permissions). The rationale:

| Requirement | Library Support | Manual |
| --- | :---: | :---: |
| Per-organization role scoping (same user, different roles in different organizations) | ❌ | ✅ |
| Runtime-configurable permissions (org admin edits permissions via UI) | ❌ | ✅ |
| Row-level scoping (`orders:own:read`) | ❌ | ✅ |
| No library deprecation risk | ❌ | ✅ |

---

## `core/permissions.py` — Full Contract

```python
# Example only — adapt to your project

# core/permissions.py

from uuid import UUID
from functools import wraps
from ninja.errors import HttpError

from apps.authorization.repositories.django_repo import DjangoAuthorizationRepository


# ─── Core Resolver ────────────────────────────────────────────────────────────

def get_permissions(user_id: UUID, organization_id: UUID) -> set[str]:
    """
    Single resolver — everything flows through here.
    Returns the full permission set for a user in an organization.
    Live DB query — no cache.
    """
    repo = DjangoAuthorizationRepository()
    return repo.get_permissions(user_id, organization_id)


def has_permission(user_id: UUID, organization_id: UUID, permission: str) -> bool:
    return permission in get_permissions(user_id, organization_id)


# ─── Django-Ninja Decorator ───────────────────────────────────────────────────

def requires_permission(permission: str):
    """
    Decorator for Django-Ninja endpoints.
    Usage: @requires_permission("orders:write")
    Resolves organization_id from path kwargs first, then falls back to request attribute.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            organization_id = kwargs.get("organization_id") or getattr(request, "organization_id", None)
            if not organization_id or not has_permission(request.user.id, organization_id, permission):
                raise HttpError(403, f"Permission required: {permission}")
            return func(request, *args, **kwargs)
        return wrapper
    return decorator
```

---

## Permission String Convention

All permission strings use explicit `(resource:action)` format. No wildcards. No `manage` shorthand — always expand to discrete verbs.

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

Any service that mutates `organization_memberships` or `role_permissions` **must** call the cache invalidation helper **after** the DB transaction commits. Never inside `transaction.atomic()`.

```python
# Example only — adapt to your project

# apps/authorization/services/membership_services/invite_member_service.py

from django.db import transaction

from apps.authorization.repositories.django_repo import DjangoAuthorizationRepository
from apps.authorization.cache.redis_repo import AuthorizationCacheRepo


class InviteMemberService:
    def execute(self, data: dict) -> dict:
        repo = DjangoAuthorizationRepository()
        cache = AuthorizationCacheRepo()

        with transaction.atomic():
            membership = repo.create_membership(data)

        # Cache invalidation AFTER commit — never inside the atomic block
        cache.invalidate(organization_id=data["organization_id"], user_id=data["user_id"])
        return membership
```

---

## Row-Level Scoping (`orders:own:read`)

When a user holds only `orders:own:read` (not the full `orders:read`), the router applies an additional filter before calling the repository. Row-level scoping logic lives in the API layer (not in the repository, not in the service).

```python
# Example only — adapt to your project

# apps/orders/api/router.py

@router.get("/organizations/{organization_id}/orders/", response=ApiResponse[PaginatedData[OrderResponseSchema]])
def list_orders(request, organization_id: UUID):
    """
    List orders for an organization.

    Users with orders:read see all orders. Users with orders:own:read see only their own.
    """
    perms = get_permissions(request.user.id, organization_id)
    repo  = DjangoOrderRepository()

    if "orders:read" in perms:
        data = repo.list_by_organization(organization_id)
        return ApiResponse(message="Orders retrieved successfully", data=data)

    if "orders:own:read" in perms:
        auth_repo = DjangoAuthorizationRepository()
        assignee_id = auth_repo.get_assignee_by_user(request.user.id, organization_id)
        data = repo.list_by_assignee(organization_id, assignee_id) if assignee_id else []
        return ApiResponse(message="Orders retrieved successfully", data=data)

    raise HttpError(403, "Permission required")
```

---

# Real-Time State Layer (Redis)

> **This section applies only to modules that maintain live, distributed state.** Purely CRUD modules (`items`, `organizations`, `users`, `authorization`) do **not** need a `cache/` folder.

## Deciding If Your Module Needs Redis

| Module needs Redis? | Criteria |
| --- | --- |
| ✅ Yes | Live counters, priority queues, real-time availability, pub/sub, TTL-based lookups |
| ❌ No | Simple CRUD, data that only changes on explicit user action, no real-time consumers |

| Module | Needs Redis? | Reason |
| --- | :---: | --- |
| `orders` | ❌ | Booking uses OCC on `time_slots` (DB-level); no live counters |
| `scheduling` | ⚠️ | Cache infrastructure exists but is disabled — preserved for future use |
| `search` | ✅ | RediSearch index for multi-organization discovery |
| `authorization` | ✅ | Permission cache (TTL 300s, key: `perm:{org_id}:{user_id}`) |
| `organizations` | ❌ | Pure CRUD |
| `users` | ❌ | Pure CRUD — rate limiting belongs at infrastructure/nginx layer |
| `catalog` | ❌ | CRUD |

---

## Redis Key Naming Convention

All Redis keys follow `<domain>:<id>:<type>`. All segments are in English.

```
# Queue module
queue:{queue_id}:counter                          → STRING  — atomic INCR
queue:{queue_id}:state                            → HASH    — live state
queue:{queue_id}:tickets                          → ZSET    — priority queue

# Slot availability
slots:{org_id}:{catalog_id}:{date}                → HASH    — TTL 60s

# Permission cache
perm:{org_id}:{user_id}                           → STRING  — TTL 300s

# Search cache
search:{org_id}:{query}:{month}                   → STRING  — TTL 3600s

# Generic patterns
<domain>:{id}:state                               → HASH    — live state snapshot
pubsub:<domain>:{id}                              → CHANNEL — entity-scoped events
session:<domain>:<code>                           → STRING  — short-lived (with TTL)
```

---

## Redis Failure Recovery

- Every Redis write must have a corresponding database write first. The SQL database is the source of truth.
- A Celery management command must exist to rebuild Redis state from the DB on reconnect.
- The rebuild command must be idempotent — safe to run multiple times without side effects.
- Never call `KEYS *` in production — use `SCAN` + `DEL` in batches.

---

# WebSocket + Pub/Sub Rules

## Event Flow (Always in this order)

```
Service → DB write → Redis state update → Redis PUBLISH → Channel consumer → WebSocket push
```

Never skip steps. The DB write always precedes the Redis write.

## Consumer Rules

- Consumers only subscribe and forward. **Zero business logic in consumers.**
- Consumers must handle disconnections gracefully — must never crash the server.
- Channel names are entity-scoped:

```python
# Example only — adapt to your project

"queue_global"              # queue module — all queue events
"item_{id}"                 # item-scoped events
"order_{id}"                # order-scoped events
"organization_{id}"         # organization-scoped events
```

---

# Cross-Module Communication

When a service or the API needs data from another module, it imports that module's `django_repo.py` directly. No injected interfaces, no adapter layers — just a plain import.

```python
# Example only — adapt to your project

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

---

# Dependency Flow Matrix

```
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

---

# Automated Enforcement (pyproject.toml)

```toml
# Example only — adapt to your project

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

---

# Architectural Rules

- **Rule 1 (Domain Purity):** The `domain/` layer must stay 100% pure: no Django imports, no ORM models, no Redis, no Celery, no HTTP concerns. Test: `pytest apps/<module>/domain/` must pass with `DJANGO_SETTINGS_MODULE` unset.

- **Rule 2 (Thin API Layer):** `router.py` contains zero business logic. For reads: import and call the repository directly. For writes: import and call the service. No orchestration beyond that single delegation.

- **Rule 3 (Services Own All Writes):** Every POST/PUT/PATCH/DELETE must go through a service class with an `execute()` method. Never write to the ORM directly from `router.py`.

- **Rule 4 (Repositories Own ORM Access):** ORM model imports (`models.py`) are strictly forbidden everywhere except `repositories/django_repo.py` and `tasks/`. Any layer that needs data goes through the repository.

- **Rule 5 (One Service, One Use-Case):** A service class has exactly one public method (`execute()`) performing exactly one use-case. If two actions are needed, that is two service classes in two files.

- **Rule 6 (No Constructor Injection):** Services and repositories are never passed as parameters to `__init__`. All dependencies are imported directly at the top of the file and instantiated at call time. No DI containers. No Protocol interfaces.

- **Rule 7 (No Django Signals):** Never use Django signals for domain side effects. Call `core/websocket.py` broadcast helpers explicitly from services, after the DB write, outside `transaction.atomic()`.

- **Rule 8 (Thin Tasks):** Celery tasks are thin shims — they instantiate a service and call `execute()`. Maximum 10 lines per task file. All logic lives in the service.

- **Rule 9 (Zero Consumer Logic):** WebSocket consumers subscribe, receive, and forward. They never call services, touch the ORM, or apply business rules.

- **Rule 10 (Exception Hierarchy):** All domain exceptions extend `core.exceptions.ApplicationError`. Services raise domain exceptions. The API maps them to HTTP status codes via a global exception handler in `config/api.py`.

- **Rule 11 (Redis After DB):** Every Redis write must be preceded by a successful database write. The SQL database is the source of truth. Redis is the speed layer. They are never out of sync for more than one operation.

- **Rule 12 (Cache Invalidation Outside Transactions):** Never call cache invalidation functions inside `transaction.atomic()`. Call them after the `with` block closes, or register them via `transaction.on_commit()`.

- **Rule 13 (No Cross-Module ORM Imports):** When a service or the router needs data from another module, it imports and uses that module's `django_repo.py`. Direct imports of another module's `models.py` are forbidden everywhere except within that module's own repository.

- **Rule 14 (Permission Resolution via Core):** All permission checks must flow through `core/permissions.py`. No service, repository, or router may directly query `organization_memberships` or `role_permissions` for authorization purposes.

- **Rule 15 (Explicit Permission Strings):** Permission strings are explicit `(resource:action)` tuples. No wildcards. No `manage` shorthand — expand to `create`, `update`, `delete` individually.

- **Rule 16 (System Roles Are Immutable):** System roles (`is_system = TRUE`) may not be deleted or have their permissions modified. Enforce this as a domain rule in `authorization/domain/rules.py`.

- **Rule 17 (Organization Must Have One Admin):** An organization must always have at least one active ADMIN member. `deactivate_member_service` and `change_role_service` must verify this invariant before committing.

- **Rule 18 (No Naming Dumping Grounds):** Folder names like `misc/`, `helpers/`, `common/`, or `other/` are strictly banned. Every folder must have a clear, bounded responsibility.

- **Rule 19 (Encapsulate Business Conditionals):** Never write loose boolean conditions inside routers or services (e.g., `user.role == 'ADMIN' and org.is_active`). Wrap them in descriptive functions in `domain/rules.py` (e.g., `can_manage_organization(user, org)`).

- **Rule 20 (Safe Deletability):** If dropping a module's folder breaks an unrelated module, your domain boundaries are bleeding. Each module must be independently removable.

- **Rule 21 (No KEYS \* in Production Redis):** Use `SCAN` + `DEL` in batches. `KEYS *` blocks the Redis event loop and is forbidden in any environment with real data.

- **Rule 22 (Search Module Is Read-Only):** The `search` module contains no services and no write repository methods. It reads from its RediSearch index only. No writes, no mutations.

- **Rule 23 (OpenAPI Documentation Is Mandatory):** Every endpoint must have a docstring, a declared response type, and a router tag. Stale or missing documentation is a bug, not a nice-to-have.

- **Rule 24 (Standardized API Responses):** All endpoints must return the `ApiResponse[T]` wrapper for successful responses. All errors must be caught by the global exception handler and formatted into the `ErrorResponse` schema. Never return raw data or ad-hoc JSON structures.

---

# Anti-Patterns Reference

| If you see this | Module context | The fix |
| --- | --- | --- |
| `ItemORM.objects.get()` inside a service | any | Move to `DjangoItemRepository`; import the repo class in the service |
| Business logic in `router.py` | any | Extract to a service class; router calls `service.execute()` |
| `router.py` writing to ORM directly | any | Create a service class — router never writes |
| Selector class anywhere | any | Selectors are removed from this architecture; reads go through repositories |
| `class BaseXxxService: def __init__(self, repo): ...` | any | Remove DI from base; services import repos directly at the top of the file |
| Repository passed as `__init__` parameter | any | Import the concrete repository class directly — no constructor injection |
| Django signals used for domain side effects | any | Call broadcast helper explicitly from the service after the DB write |
| Task with more than 10 lines | any | Extract logic to service; task is a thin shim |
| `broadcast()` inside `transaction.atomic()` | any | Call after the `with` block closes, or use `transaction.on_commit()` |
| One service file handling two unrelated use-cases | any | Split into two files, one per use-case |
| Strategy behaviour hardcoded with `if/elif` in a service | any | Create a strategy subclass in `domain/rules.py` |
| Permission check duplicated in service AND router | any | Checks live only in `core/permissions.py` via decorator |
| `from apps.items.models import ItemORM` inside `orders/` | orders | Import `DjangoItemRepository` and use its read methods |
| `from apps.authorization.models import RoleORM` in any service | any | Import `DjangoAuthorizationRepository` instead |
| Global role assigned with no `organization_id` scope | authorization | All roles are per-organization; `organization_id` is always NOT NULL |
| `cache.keys("search:*")` or `KEYS search:*` in production | search | Use `SCAN` + `DEL` in batches |
| Cache invalidation inside `transaction.atomic()` | any | Move after the `with` block closes or register via `on_commit` |
| `search` module importing a write repository or service | search | The `search` module is read-only by Rule 22 |
| Endpoint missing a docstring or response type | any | Add docstring and `response=ApiResponse[T]` annotation per Rule 23 |
| Endpoint returning raw data or un-wrapped dict | any | Wrap response in `ApiResponse(data=...)` per Rule 24 |
| Raising `HttpError` directly for domain business rules | any | Raise a domain exception subclassing `ApplicationError`; let the global handler format the `ErrorResponse` |

---

# Final Architectural Performance Metrics

| Area | Result | Target Metric Met |
| --- | --- | --- |
| **Scalability** | High | Unlocked parallel workflows for multi-team execution. |
| **Maintainability** | High | Code changes stay contained inside module directories. |
| **Team Collaboration** | Excellent | Minimized codebase conflicts via clear domain boundaries. |
| **Domain Isolation** | Strong | Core business rules are protected from ORM and HTTP churn. |
| **Refactoring Safety** | High | Isolated scopes make code deletion safe and reliable. |
| **Feature Ownership** | Clear | Modules track perfectly to business domains and engineering teams. |
| **Testing** | Easier | Pure domain layers accept unit testing without any Django setup. |
| **Cognitive Load** | Lower | Developers only need to reason about one module directory at a time. |
| **Microservice Readiness** | Good | Modules are prepared for future decomposition into standalone services. |
| **Simplicity** | Excellent | No DI containers, no injected interfaces, no selector indirection — direct imports reduce mental overhead without sacrificing layer clarity. |
| **API Discoverability** | Excellent | OpenAPI documentation is enforced on every endpoint, ensuring the spec is always accurate and up to date. |
| **Client Integration** | Excellent | Standardized response structures eliminate special-casing on the frontend, reducing integration bugs and parsing logic. |
