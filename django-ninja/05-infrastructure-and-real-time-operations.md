# Guide 5: Infrastructure & Real-Time Operations

> **Disclaimer:** All code snippets, module names, class names, and examples throughout this document are **illustrative examples only**. Adapt them to your specific business domain, project requirements, and team conventions.

## Real-Time State Layer (Redis)

> **This section applies only to modules that maintain live, distributed state.** Purely CRUD modules (`items`, `organizations`, `users`, `authorization`) do **not** need a `cache/` folder.

### Deciding If Your Module Needs Redis

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

### Redis Key Naming Convention

All Redis keys follow `<domain>:<id>:<type>`. All segments are in English.

```text
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

### Redis Failure Recovery

- Every Redis write must have a corresponding database write first. The SQL database is the source of truth.
- A Celery management command must exist to rebuild Redis state from the DB on reconnect.
- The rebuild command must be idempotent — safe to run multiple times without side effects.
- Never call `KEYS *` in production — use `SCAN` + `DEL` in batches.

## WebSocket + Pub/Sub Rules

### Event Flow (Always in this order)

```text
Service → DB write → Redis state update → Redis PUBLISH → Channel consumer → WebSocket push
```

Never skip steps. The DB write always precedes the Redis write.

### Consumer Rules

- Consumers only subscribe and forward. **Zero business logic in consumers.**
- Consumers must handle disconnections gracefully — must never crash the server.
- Channel names are entity-scoped:

```python
"queue_global"              # queue module — all queue events
"item_{id}"                 # item-scoped events
"order_{id}"                # order-scoped events
"organization_{id}"         # organization-scoped events
```

## Asynchronous Tasks (Celery)

Task files live in the `tasks/` directory of a module. They are thin wrappers—maximum 10 lines per task file. They import the service class and call `execute()`. They never contain business logic, ORM queries, or multi-step orchestration.

```python
# apps/orders/tasks/process_order_task.py

from apps.orders.services.order_services.process_order_service import ProcessOrderService


@shared_task
def process_order_task(order_id: str):
    service = ProcessOrderService()
    service.execute(order_id)
```

