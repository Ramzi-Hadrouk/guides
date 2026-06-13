# Guide 3: Domain Logic & Write Operations

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions.

## Service Layer — Write Orchestration

Services own all write use-cases. They import repositories directly via normal Python imports — no constructor injection, no interface parameters.

```python
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

## Repository Layer — Data Access

The repository is the **only place that touches the ORM**. It owns all read and write queries for its module and maps between ORM models and domain entities. Read methods and write methods are clearly sectioned inside the same class.

```python
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

