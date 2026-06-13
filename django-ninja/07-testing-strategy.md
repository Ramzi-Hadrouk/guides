# Guide 7: Testing Strategy

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions.

Testing is strictly分层 into three categories based on scope and speed: **Unit**, **Integration**, and **API**. Each test type has a specific home in the module directory and strict rules about what it may and may not touch.

## Directory Structure

Every module contains a `tests/` directory with three sub-folders:

```txt
apps/<module>/
└── tests/
    ├── unit/               # Domain + service unit tests (no DB, no HTTP)
    ├── integration/        # Service-to-DB flow tests (Real DB, no HTTP)
    └── api/                # Endpoint tests (HTTP client, Real DB)
```

## 1. Unit Tests (`tests/unit/`)

Fast, isolated tests with zero I/O. They validate domain logic, business rules, and service orchestration assuming dependencies behave correctly.

**Rules:**
- **No database access** (Use `@pytest.mark.django_db(databases=[])` or mock repositories).
- **No network access**.
- Service tests must mock repository calls to test orchestration logic (e.g., transaction boundaries, exception raising).
- Domain rule tests test pure functions.

**Naming Convention:** `test_<feature>_<layer>.py` (e.g., `test_order_service_unit.py`, `test_priority_rules_unit.py`)

**Example: Testing a Service with a Mocked Repository**
```python
# apps/orders/tests/unit/test_order_service_unit.py

from unittest.mock import MagicMock
from apps.orders.services.order_services.create_order_service import CreateOrderService
from apps.orders.domain.exceptions import ItemNotFoundError


def test_create_order_raises_error_if_item_not_found():
    # Arrange
    mock_item_repo = MagicMock()
    mock_item_repo.get_by_id.return_value = None  # Simulate item not existing

    mock_order_repo = MagicMock()

    service = CreateOrderService()
    # Override direct imports for testability if needed, or use patch
    # In this architecture, direct imports are standard, so we patch them:
    with patch("apps.orders.services.order_services.create_order_service.DjangoItemRepository", return_value=mock_item_repo), \
         patch("apps.orders.services.order_services.create_order_service.DjangoOrderRepository", return_value=mock_order_repo):

        # Act & Assert
        with pytest.raises(ItemNotFoundError):
            service.execute({"item_id": "123", "organization_id": "456"})
```

## 2. Integration Tests (`tests/integration/`)

Full-stack tests against the real database to verify that repositories map data correctly and service transactions commit as expected.

**Rules:**
- **Must use the real database** (Usually a test SQLite or test PostgreSQL DB).
- **No HTTP Client**. Call services or repositories directly.
- Must roll back data between tests (use Django's `TestCase` or `pytest-django` transactions).
- Validate ORM-to-Domain mapping and database constraints (e.g., unique constraints).

**Naming Convention:** `test_<feature>_<layer>.py` (e.g., `test_order_repo_integration.py`)

**Example: Testing a Repository Mapping**
```python
# apps/items/tests/integration/test_item_repo_integration.py

import pytest
from apps.items.models import ItemORM
from apps.items.repositories.django_repo import DjangoItemRepository


@pytest.mark.django_db
def test_save_and_retrieve_item():
    # Arrange
    repo = DjangoItemRepository()
    data = {"name": "Test Item", "description": "A test", "organization_id": "123"}

    # Act
    saved_item = repo.save(data)
    retrieved_item = repo.get_by_id(saved_item.id)

    # Assert
    assert retrieved_item is not None
    assert retrieved_item.name == "test item"  # Repo logic lowercases names
    assert isinstance(retrieved_item, Item)     # Ensures mapping to domain entity
```

## 3. API Tests (`tests/api/`)

End-to-end tests for the HTTP boundary. They ensure that routers accept valid input, reject invalid input, apply permissions, and return the correct `ApiResponse` / `ErrorResponse` JSON structures.

**Rules:**
- **Must use the Django test client** (or Django-Ninja's `TestClient`).
- **Must hit the HTTP endpoint** just like a frontend client would.
- **Uses the real database**.
- Tests authentication, authorization decorators, Pydantic validation, and response serialization.
- Does not mock services or repositories (this is a full vertical slice test).

**Naming Convention:** `test_<feature>_<layer>.py` (e.g., `test_order_router_api.py`)

**Example: Testing an API Endpoint**
```python
# apps/items/tests/api/test_item_router_api.py

import pytest
from django.test import Client
from apps.users.models import UserORM
from apps.organizations.models import OrganizationORM


@pytest.mark.django_db
def test_create_item_api_success():
    # Arrange
    client = Client()
    user = UserORM.objects.create(username="testuser")
    org = OrganizationORM.objects.create(name="Test Org", owner_user_id=user.id)

    # Act
    response = client.post(
        f"/api/organizations/{org.id}/items/",
        content_type="application/json",
        HTTP_AUTHORIZATION="Bearer <token>",  # Mock auth as needed
        data={"name": "New Item", "description": "From API test"}
    )

    # Assert
    assert response.status_code == 200
    data = response.json()
    
    # Validate standard response structure
    assert data["success"] is True
    assert data["message"] == "Item created successfully"
    assert data["data"]["name"] == "new item"
    assert "id" in data["data"]


@pytest.mark.django_db
def test_create_item_api_validation_error():
    # Arrange
    client = Client()
    
    # Act (Missing 'name' field which is required by schema)
    response = client.post(
        "/api/organizations/123/items/",
        content_type="application/json",
        data={"description": "Missing name"}
    )

    # Assert
    assert response.status_code == 422
    data = response.json()
    assert data["success"] is False
    assert data["message"] == "Validation failed"
    assert any(err["field"] == "name" for err in data["errors"])
```

## Testing Rules Summary

1. **Unit tests** are for logic. They are fast and require zero infrastructure. Mock all I/O.
2. **Integration tests** are for data. They verify the database boundary and ORM mapping. Use a real DB, no HTTP.
3. **API tests** are for the contract. They verify the HTTP boundary, status codes, JSON shapes, and permissions. Use a real DB and the HTTP client.
4. Never write unit tests for Django ORM models or Pydantic schemas directly; test them implicitly via integration and API tests.
5. All tests must be deterministic. Never rely on external APIs or live Redis instances.