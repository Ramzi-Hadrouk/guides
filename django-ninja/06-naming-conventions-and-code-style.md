# Guide 6: Naming Conventions & Code Style

> **Disclaimer:** All naming across the entire codebase must be in **English**. This ensures consistency, readability for international teams, and alignment with Python and Django ecosystem conventions.

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

## Method Naming Rules

- All method names use **`snake_case`**.
- Repository read methods: **`get_by_<field>`**, **`list_by_<field>`**, **`exists_by_<field>`**
  - `get_by_id()`, `list_by_organization()`, `exists_by_email()`
- Repository write methods: **`save`**, **`update`**, **`delete`**, **`bulk_create`**
- Service public method: **`execute`** — always this exact name; the class name carries the intent
- Domain rule functions: **`can_<action>`**, **`calculate_<result>`**, **`validate_<condition>`**
  - `can_cancel_order()`, `calculate_priority()`, `validate_membership()`
- Mapper methods (private): **`_map_to_domain`**, **`_map_to_orm`**

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

## URL / Endpoint Naming Rules

- URL segments use **`kebab-case`**.
- Plural nouns for collections: **`/api/v1/orders/`**
- Nested resources: **`/api/v1/organizations/{org_id}/members/`**
- Specific actions on a resource: **`/api/v1/orders/{order_id}/cancel/`**

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

## Schema Field Naming Rules

- All schema field names use **`snake_case`**.
- Follow the same rules as variable naming above.
- Examples: `order_id`, `status`, `created_at`, `is_active`, `duration_min`

## ORM Model Field Naming Rules

- All field names use **`snake_case`**.
- Follow the same rules as variable naming above.
- Foreign key fields: **`<entity>_id`** (e.g., `organization_id`, `user_id`)
- Examples: `ticket_number`, `status`, `created_at`, `is_active`, `duration_min`

