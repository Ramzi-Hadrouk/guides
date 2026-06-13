# Guide 4: Security & Authorization (RBAC)

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions.

> **This section governs `core/permissions.py` and the `authorization` module.** The permission decorator touches every API endpoint in every module, so the contract here is non-negotiable.

## Architecture Decision: Manual Tenant-Scoped RBAC

The platform uses manual RBAC instead of a third-party library (django-guardian, django-rules, django-role-permissions). The rationale:

| Requirement | Library Support | Manual |
| --- | :---: | :---: |
| Per-organization role scoping (same user, different roles in different organizations) | ❌ | ✅ |
| Runtime-configurable permissions (org admin edits permissions via UI) | ❌ | ✅ |
| Row-level scoping (`orders:own:read`) | ❌ | ✅ |
| No library deprecation risk | ❌ | ✅ |

## `core/permissions.py` — Full Contract

```python
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

## Permission String Convention

All permission strings use explicit `(resource:action)` format. No wildcards. No `manage` shorthand — always expand to discrete verbs.

```text
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

## Cache Invalidation Contract

Any service that mutates `organization_memberships` or `role_permissions` **must** call the cache invalidation helper **after** the DB transaction commits. Never inside `transaction.atomic()`.

```python
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

## Row-Level Scoping (`orders:own:read`)

When a user holds only `orders:own:read` (not the full `orders:read`), the router applies an additional filter before calling the repository. Row-level scoping logic lives in the API layer (not in the repository, not in the service).

```python
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

