# Guide 2: API Design & Contracts

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions.

## Standardized API Response Structure

All API endpoints must return a unified, predictable JSON structure for both successful requests and errors. This ensures frontend clients, mobile apps, and third-party integrations can parse responses consistently without special-casing endpoints.

### Core Response Schemas

The standard response envelopes are defined in `core/responses.py` using Pydantic generics. This allows type-safe payloads while maintaining a uniform outer shell.

```python
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

### Success Response Contract

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

### Error Response Contract

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

### Global Exception Handler

To enforce the error response structure automatically, register a global exception handler in `config/api.py`. Domain exceptions are caught and mapped to the `ErrorResponse`.

```python
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

### Base Domain Exception

Update `core/exceptions.py` to support the fields required by the error handler:

```python
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

## OpenAPI Documentation

All endpoints must be documented using Django-Ninja's built-in OpenAPI support. The OpenAPI spec is the single source of truth for API consumers and must stay accurate.

### Root API Configuration

Configure the `NinjaAPI` instance in `config/api.py` with meaningful metadata:

```python
from ninja import NinjaAPI

api = NinjaAPI(
    title="Platform API",
    version="1.0.0",
    description="API documentation for the platform. All endpoints require authentication unless explicitly marked as public.",
    docs_url="/docs/",
    openapi_url="/openapi.json",
)
```

### Router Tags

Every module's router must declare a **tag** that groups its endpoints in the OpenAPI UI. Tags use the module's English name in plural form, `PascalCase` for display.

```python
from ninja import Router

router = Router(tags=["Orders"])
```

Register the router with the same tag in `config/api.py`:

```python
from config.api import api
from apps.orders.api.router import router as orders_router

api.add_router("/orders/", orders_router, tags=["Orders"])
```

### Endpoint Docstrings

Every endpoint function must have a **docstring** that describes what it does. The first line is the summary; subsequent lines are the description.

```python
@router.get("/orders/{order_id}/", response=ApiResponse[OrderResponseSchema])
def get_order(request, order_id: UUID):
    """
    Retrieve a single order by ID.

    Returns the full order details including items, status, and timestamps.
    Raises 404 if the order does not exist.
    """
    ...
```

### Response and Request Annotations

All endpoints must declare their `response` type using the `ApiResponse[T]` wrapper schema. For `POST`/`PUT`/`PATCH`, the payload schema must be explicitly typed as a parameter.

```python
@router.post("/orders/", response=ApiResponse[OrderResponseSchema])
def create_order(request, payload: CreateOrderRequestSchema):
    """Create a new order."""
    ...


@router.get("/orders/", response=ApiResponse[PaginatedData[OrderResponseSchema]])
def list_orders(request, organization_id: UUID):
    """List all orders for a given organization."""
    ...
```

### Operation IDs

For endpoints that need stable, client-facing operation IDs (e.g., code generation), use the `operation_id` parameter:

```python
@router.get("/orders/{order_id}/", response=ApiResponse[OrderResponseSchema], operation_id="getOrder")
def get_order(request, order_id: UUID):
    """Retrieve a single order by ID."""
    ...
```

### Deprecated Endpoints

When an endpoint is being phased out, mark it with `deprecated=True` instead of removing it immediately:

```python
@router.get("/orders/legacy/list/", response=ApiResponse[PaginatedData[OrderResponseSchema]], deprecated=True)
def list_orders_legacy(request, organization_id: UUID):
    """
    [DEPRECATED] Use GET /orders/ instead.

    Legacy list endpoint retained for backward compatibility.
    Will be removed in v2.
    """
    ...
```

### OpenAPI Rules Summary

1. **Every endpoint must have a docstring** — first line is the summary, the rest is the description.
2. **Every router must declare a tag** matching its module name.
3. **Every endpoint must declare its `response` type** via the `response` parameter, wrapped in `ApiResponse[T]` or `ApiResponse[PaginatedData[T]]`.
4. **Every `POST`/`PUT`/`PATCH` payload must be a typed schema parameter**, never raw `dict` or `Request`.
5. **Use `operation_id`** when the auto-generated ID is not stable or readable enough for client generation.
6. **Mark deprecated endpoints with `deprecated=True`**, do not silently remove them.
7. **Keep the docstring in sync with the endpoint behavior** — stale docs are worse than no docs.

## API Layer — HTTP Controllers

The API layer is **thin by law**. It delegates immediately — never implements. For reads (GET), it imports and calls the repository. For writes (POST/PUT/PATCH/DELETE), it imports and calls the service. All return types use the standardized `ApiResponse` wrapper.

```python
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

