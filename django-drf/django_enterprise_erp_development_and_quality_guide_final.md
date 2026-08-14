# Django Enterprise ERP Architecture — Guide 2: Development Standards & Quality

**Mandatory engineering standard for implementation, maintenance, refactoring, and AI-assisted development.**

This guide defines how features are implemented inside the architecture established by **Guide 1 — Project Preparation & Configuration**. It is intended for the development team and AI coding agents.

A feature is not considered complete merely because it works or tests pass. It must respect the module boundaries, business invariants, API contract, database integrity, security, concurrency, observability, and quality rules below.

## Reference architecture at a glance

A developer or AI coding agent should be able to understand the intended architecture from this example **without reading Guide 1 first**.

The project uses a **Django-native modular monolith**:

- Django models are the business entities and persistence representation.
- Django ORM is the data-access layer.
- Use cases contain application/business orchestration.
- Serializers define API input/output contracts.
- Pure business rules are kept as small functions when they do not need database access.
- No repository/DAO layer is required by default.
- Events, Celery, caching, and other infrastructure are introduced only when a concrete requirement exists.

```text
apps/ventes/
├── __init__.py
├── apps.py
├── models.py                         # Django entities + persistence
├── admin.py
├── urls.py
├── migrations/
│
├── business/                         # optional pure business logic
│   ├── facture_rules.py              # pure calculations/invariants
│   ├── remise_rules.py
│   ├── exceptions.py                 # business conflicts
│   └── constants.py
│
├── crud/                             # only for genuinely simple model CRUD
│   └── mode_paiement/
│       ├── serializer.py
│       ├── view.py
│       └── permissions.py
│
├── usecases/
│   ├── creer_facture/
│   │   ├── service.py                # one business operation
│   │   ├── serializer.py             # API input/output schema
│   │   ├── view.py                   # HTTP boundary
│   │   └── permissions.py
│   │
│   ├── valider_facture/
│   │   └── ...
│   │
│   ├── annuler_facture/
│   │   └── ...
│   │
│   ├── obtenir_facture/
│   │   ├── service.py                # read/query operation
│   │   ├── serializer.py
│   │   ├── view.py
│   │   └── permissions.py
│   │
│   └── lister_factures/
│       └── ...
│
├── tasks/                            # only when async work is justified
│
├── events/                           # optional; only when durable/event-driven
│   └── ...
│
└── tests/
    ├── unit/
    │   ├── business/
    │   └── usecases/
    │       ├── creer_facture/
    │       ├── valider_facture/
    │       └── annuler_facture/
    ├── integration/                  # PostgreSQL, ORM, constraints, transactions
    └── api/                          # DRF/API contract tests
```

### The normal request flows

Simple model-backed CRUD:

```text
HTTP
 ↓
ViewSet / Generic View
 ↓
ModelSerializer
 ↓
Django ORM
 ↓
PostgreSQL
```

Business write:

```text
HTTP
 ↓
View
 ↓
Input Serializer
 ↓
Use-case service
 ↓
Business rules
 ↓
Django Model / ORM
 ↓
PostgreSQL
```

Business read:

```text
HTTP
 ↓
View
 ↓
Use-case service
 ↓
QuerySet / ORM
 ↓
Output Serializer
 ↓
JSON
```

Pure calculation:

```text
Use-case service
 ↓
business rule function
 ↓
value
```

Asynchronous side effect, only when needed:

```text
Committed transaction
 ↓
on_commit / outbox
 ↓
Celery task
 ↓
use-case or integration operation
```

### Non-negotiable boundaries

1. **Models are entities.** Do not introduce a second entity layer merely to imitate another architecture.
2. **The ORM is the persistence layer.** Do not create repositories/DAOs by default.
3. **Serializers are API schemas.** They validate and represent data; they do not own business workflows.
4. **Use-case services orchestrate operations.** They may read/write Django models and use transactions.
5. **Business rule functions stay pure when possible.** They should not query the database or call external systems unless that dependency is inherently part of the rule.
6. **Views stay thin.**
7. **Simple CRUD stays simple.** Do not turn every CRUD endpoint into a use case.
8. **Infrastructure is optional.** Events, Celery, Redis, search systems, and extra abstractions require a concrete reason.

---


## Index

- Chapter 2 — Development
  - 2.1 Use case structure and simple-model CRUD
  - 2.2 Write use case rules
  - 2.3 Read use case rules
  - 2.4 Business rules and invariants
  - 2.5 Exceptions and error taxonomy
  - 2.6 Serializers
  - 2.7 Views
  - 2.8 Permissions and authorization
  - 2.9 Models and persistence
  - 2.10 OOP vs functions
  - 2.11 Transactions and concurrency
  - 2.12 Database constraints and integrity
  - 2.13 Query performance
  - 2.14 Pagination, filtering, sorting, and search
  - 2.15 Background jobs
  - 2.16 Events, outbox, and side effects
  - 2.17 Signals and save()
  - 2.18 Document numbering and idempotency
  - 2.19 Files and external integrations
  - 2.20 Caching
  - 2.21 Observability and auditability
  - 2.22 Testing
  - 2.23 API versioning and compatibility
  - 2.24 Data migrations and deployment safety
  - 2.25 AI agent rules and borders
- Chapter 3 — Strict Quality Checklist

---

# Chapter 2 — Development

## 2.1 Use case structure and simple-model CRUD

Use a use case when an operation has meaningful business behavior.

A use case folder normally contains:

```text
usecases/valider_facture/
├── service.py
├── serializer.py
├── view.py
└── permissions.py
```

Not every operation needs every file.

For a simple read, a minimal structure is valid:

```text
usecases/obtenir_facture/
├── service.py
├── serializer.py
└── view.py
```

### Simple model CRUD

For simple models such as `ModePaiement` or `TvaRate`, use a `crud/` folder with standard DRF generic views or `ModelViewSet`.

```text
crud/
└── mode_paiement/
    ├── serializer.py
    ├── view.py
    └── permissions.py
```

Use `ModelSerializer` here when the API maps naturally to the model.

Do not create a business use case for ordinary create/update/delete merely for architectural consistency.

The rule is:

```text
Simple model CRUD → crud/
Business operation → usecases/
```
## 2.2 Write use case rules

A write use case normally follows:

```text
load
→ authorize
→ validate business state
→ apply business rules
→ mutate models
→ persist atomically
→ publish post-commit side effects when needed
→ return result
```

A service may be a plain function. A class is not required.

Example:

```python
@transaction.atomic
def valider_facture(facture_id, utilisateur):
    facture = (
        Facture.objects
        .select_for_update()
        .select_related("client")
        .get(id=facture_id)
    )

    verifier_autorisation(utilisateur, facture)
    verifier_facture_modifiable(facture)

    facture.statut = StatutFacture.VALIDEE
    facture.save(update_fields=["statut", "date_modification"])

    transaction.on_commit(
        lambda: publier_facture_validee(facture.id)
    )

    return facture
```

The service orchestrates the operation.

Pure rule functions make individual decisions/calculations.

Django models represent the entities being changed.

The database enforces database-level integrity.

Keep one business operation per use-case module. Avoid giant `FactureService` classes containing unrelated workflows.
## 2.3 Read use case rules

Do not use write architecture for every GET endpoint.

### Simple read

A simple read may be:

```text
View
 ↓
QuerySet
 ↓
Serializer
```

The queryset may contain:

- `select_related()`;
- `prefetch_related()`;
- annotations;
- filtering;
- ordering;
- pagination.

### Complex read

Use a read use-case service when query composition or aggregation is substantial.

```text
GET /rapports/ventes/mensuel
        ↓
RapportVentesService
        ↓
PostgreSQL aggregation
        ↓
Serializer
```

A read service may return model instances, annotated querysets, or explicit data structures. Do not introduce a repository or selector abstraction unless a concrete problem requires it.
## 2.4 Business rules and invariants

Separate three kinds of validation.

### Input validation

Question:

> Is this request structurally valid?

Examples:

- required field missing;
- invalid UUID;
- malformed date;
- negative quantity when the API contract forbids it.

Handled primarily by serializers and basic model/database constraints.

### Business validation

Question:

> Is this operation allowed according to business rules?

Examples:

- a validated invoice cannot be edited;
- stock cannot become negative;
- a payment cannot exceed the allowed allocation;
- a closed accounting period cannot accept new entries.

Handled by business rules/use cases and backed by database integrity where possible.

### Authorization

Question:

> Is this user allowed to perform this action?

Handled by authentication, permissions, and object-level authorization.

Do not mix these three concepts unnecessarily.

### Pure calculations

Pure calculations should be functions.

```python
def calculer_total_ht(lignes):
    ...


def calculer_tva(base, taux):
    ...
```

### Stateful business concepts

Use classes when a concept genuinely has state, identity, lifecycle, or polymorphic behavior.

Do not create classes simply to hold one calculation.

---

## 2.5 Exceptions and error taxonomy

Use exceptions primarily for **business conflicts and exceptional business states**, not every possible validation error.

### Recommended categories

#### Validation

```text
VALIDATION_ERROR
```

For malformed or structurally invalid requests.

#### Not found

```text
NOT_FOUND
```

For a resource that does not exist or should not be disclosed.

#### Authorization

```text
FORBIDDEN
```

For an authenticated user without sufficient permission.

#### Business conflict

Examples:

```text
FACTURE_DEJA_VALIDEE
STOCK_INSUFFISANT
PERIODE_COMPTABLE_FERMEE
PAIEMENT_DEJA_ENREGISTRE
```

Typically mapped to HTTP 409 where appropriate.

### Business exception example

```python
class FactureDejaValideeException(BusinessError):
    code = "FACTURE_DEJA_VALIDEE"
    message = "La facture ne peut plus être modifiée."
    status = 409
```

Do not create dozens of exception classes for ordinary serializer validation.

---

## 2.6 Serializers

Serializers are the API boundary. They validate incoming data and represent outgoing data.

They are **not required to know the database**, and they must never become the business workflow.

### Plain `Serializer`

Use `serializers.Serializer` for business commands, composite payloads, or API contracts that should not be tightly coupled to one model.

```python
class CreerFactureSerializer(serializers.Serializer):
    client_id = serializers.UUIDField()
    lignes = serializers.ListField()
    remise = serializers.DecimalField(
        max_digits=12,
        decimal_places=2,
        required=False,
    )
```

The serializer validates the request:

```python
serializer = CreerFactureSerializer(data=request.data)
serializer.is_valid(raise_exception=True)

facture = creer_facture(
    **serializer.validated_data,
)
```

The use case performs the database work.

### `ModelSerializer`

Use `ModelSerializer` for simple model-backed CRUD where its automatic model integration reduces code.

```python
class ModePaiementSerializer(serializers.ModelSerializer):
    class Meta:
        model = ModePaiement
        fields = ["id", "code", "libelle"]
```

This is preferred for ordinary CRUD.

Do not use `ModelSerializer` as an excuse to place multi-step business logic into `create()` or `update()`.

### Choosing between them

```text
Simple CRUD
→ ModelSerializer

Business command / composite input
→ Serializer + use case
```

### Serializer database access

Framework-level database lookups required for declared API validation may be acceptable.

Do not use serializers for business-purpose querying or side effects.

Forbidden:

- hidden business workflows;
- external API calls;
- sending emails;
- generating PDFs;
- mutating multiple aggregates;
- changing business state during output serialization;
- arbitrary queries hidden inside `SerializerMethodField`;
- N+1 queries caused by representation logic.

Read serializers should normally receive already-fetched/optimized data.

### Responsibility boundary

```text
Serializer
→ API shape, validation, representation

Use case
→ business operation, authorization coordination, transaction, mutation

Django Model / ORM
→ entities, persistence, database interaction
```

Do not create repository/DAO layers merely to stop serializers from touching Django models.
## 2.7 Views

A view should normally do only:

1. HTTP binding;
2. authentication and permission handling;
3. serializer validation;
4. use-case invocation or query execution;
5. response construction.

Example shape:

```python
class ValiderFactureView(APIView):

    permission_classes = [ValiderFacturePermission]

    def post(self, request, facture_id):
        serializer = ValiderFactureSerializer(
            data={"facture_id": facture_id}
        )
        serializer.is_valid(raise_exception=True)

        facture = ValiderFacture().execute(
            facture_id=facture_id,
            utilisateur=request.user,
        )

        return api_success(
            data=FactureResponseSerializer(facture).data,
            code="FACTURE_VALIDEE",
        )
```

Forbidden in views:

- multi-step business workflows;
- direct manipulation of several unrelated models;
- email sending;
- PDF generation;
- long-running external calls;
- arbitrary unbounded queries;
- manual construction of inconsistent error responses.

---

## 2.8 Permissions and authorization

Authentication answers:

> Who is this user?

Authorization answers:

> What may this user do?

These are separate concerns.

Authorization should consider:

- role;
- permission;
- object ownership or organizational scope where relevant;
- workflow state;
- special approval rights.

### Never trust the frontend

The frontend may hide a button, but the backend must still enforce the rule.

Bad:

```python
if request.data["role"] == "admin":
    ...
```

Good:

```python
permission = request.user.has_perm("ventes.valider_facture")
```

For object-level checks:

```python
check_facture_access(request.user, facture)
```

A client-supplied user ID, role, permission, or organizational identifier must not be treated as authoritative.

---

## 2.9 Models and persistence

Django models are the project's **business entities and persistence representation**.

Do not introduce separate entity classes or repository/DAO layers unless a concrete requirement justifies them.

### Good model responsibilities

- fields and relationships;
- database constraints;
- indexes;
- lifecycle fields;
- narrowly scoped behavior intrinsic to the entity;
- model metadata.

A model may expose a small invariant-preserving method when that behavior naturally belongs to the entity.

Avoid turning a model into a complete application workflow.

Bad:

```python
class Facture(models.Model):
    def valider_et_envoyer_et_comptabiliser_et_notifier(...):
        ...
```

That workflow belongs in a use case.

### Constraints

Use PostgreSQL/Django constraints for invariants the database can enforce directly.

```python
models.CheckConstraint(
    check=Q(montant__gte=0),
    name="montant_non_negatif",
)
```

```python
models.UniqueConstraint(
    fields=["numero_facture"],
    name="numero_facture_unique",
)
```

### Historical records

ERP systems contain historical evidence.

Do not casually delete:

- validated invoices;
- accounting entries;
- stock movements;
- payments;
- audit records.

Use cancellation, reversal, correction, or archival workflows according to the business domain.

### Monetary values

Use `Decimal` / PostgreSQL `NUMERIC`.

Never use binary floating-point for money.
## 2.10 OOP vs functions

Use functions by default for:

- pure calculations;
- normalization;
- validation rules;
- small deterministic transformations;
- simple use-case orchestration when state is not needed.

Use a class when:

- it represents a real business concept with state;
- stateful behavior is required;
- polymorphism is genuinely useful;
- framework integration requires a class.

A use-case file named `service.py` does **not** mean the service must be a class.

Avoid generic classes such as:

```text
CommonService
BaseService
UtilityService
FactureHelper
GenericManager
```

Create abstractions only when they solve a concrete problem.
## 2.11 Transactions and concurrency

A business operation that changes multiple pieces of related state should normally execute inside one transaction.

Example:

```python
with transaction.atomic():
    facture = (
        Facture.objects
        .select_for_update()
        .get(id=facture_id)
    )

    verifier_facture_modifiable(facture)
    facture.statut = StatutFacture.VALIDEE
    facture.save()
```

### Use `select_for_update()` when necessary

Use row locking when concurrent requests could violate an invariant.

Typical examples:

- stock quantity;
- payment allocation;
- balances;
- document numbering;
- status transitions;
- counters;
- reservation quantities.

Do not use row locks indiscriminately. Locks reduce concurrency and can create deadlocks when ordering is inconsistent.

### Lock ordering

When a transaction locks several resource types, establish a consistent locking order.

For example:

```text
Document
→ Customer balance
→ Stock
→ Ledger
```

or another clearly defined project-wide ordering.

Consistency reduces deadlock risk.

### External calls

Do not perform slow external HTTP calls inside the database transaction unless absolutely unavoidable.

Prefer:

```text
DB transaction
    ↓
commit
    ↓
on_commit / outbox
    ↓
external side effect
```

---

## 2.12 Database constraints and integrity

Important business invariants should be defended at multiple levels when practical.

### Application level

Explains and executes the business workflow.

### Database level

Prevents corruption even if another code path makes a mistake.

Use:

- `ForeignKey`;
- `UniqueConstraint`;
- `CheckConstraint`;
- PostgreSQL exclusion constraints where relevant;
- non-null constraints;
- appropriate indexes.

### Example

Suppose a payment allocation must never exceed the payment amount.

Do not rely only on:

```python
if total_allocated <= payment.amount:
    ...
```

because two concurrent requests can both pass the test.

Instead:

```text
transaction
+
row lock
+
re-check
+
database invariant where practical
```

### Referential integrity

Foreign keys should be real database foreign keys whenever possible.

Do not replace relational integrity with “we check it in Python”.

---

## 2.13 Query performance

Every endpoint must be reviewed for query behavior.

Mandatory checks:

- no N+1 queries;
- `select_related()` for foreign keys and one-to-one relations;
- `prefetch_related()` for collections and many-to-many relations;
- pagination for collection endpoints;
- no unbounded `.all()` in request paths;
- use `exists()` when only existence is required;
- use `values()` / `values_list()` when full model objects are unnecessary;
- push large aggregations to PostgreSQL;
- use indexes based on actual query patterns;
- inspect query plans for expensive reporting queries.

### Do not optimize blindly

Do not add indexes because they sound useful.

Indexes have costs:

- storage;
- write amplification;
- vacuum work;
- planning complexity.

Add indexes based on real filtering, ordering, joining, uniqueness, or reporting requirements.

### Query budgets

Critical endpoints SHOULD have query-count and latency tests where regressions would be costly.

Example target:

```text
GET /factures/<id>
```

should have a stable query count rather than silently growing as serializers evolve.

---

## 2.14 Pagination, filtering, sorting, and search

Collection endpoints must have explicit limits.

Never expose an API that allows:

```text
GET /factures?limit=1000000
```

without a controlled policy.

### Pagination

Use one project-wide pagination convention.

Choose page-number, limit/offset, or cursor pagination based on the data characteristics.

Cursor pagination is particularly useful for large append-heavy datasets.

### Filtering

Whitelist supported filter fields.

Do not build arbitrary ORM filters from unsanitized query parameters.

### Sorting

Whitelist ordering fields.

Never accept unrestricted SQL fragments from the client.

### Search

For small datasets, PostgreSQL filtering may be enough.

For large search requirements, use PostgreSQL full-text search or a dedicated search system only when measurement justifies it.

Do not introduce Elasticsearch/RediSearch merely because the ERP contains a search box.

---

## 2.15 Background jobs

Use Celery or equivalent asynchronous infrastructure for:

- long-running work;
- heavy PDF generation;
- bulk imports/exports;
- email delivery;
- expensive report generation;
- external synchronization;
- scheduled maintenance;
- retryable background operations.

Do not use background jobs simply to avoid writing a clean synchronous operation.

### Task rules

Tasks must be:

- idempotent where possible;
- bounded by explicit timeouts;
- observable;
- retryable only when the operation is safe to retry;
- explicit about retry count and backoff.

Bad:

```python
def invoice_task(invoice_id):
    # 500 lines of business logic
```

Good:

```text
Celery task
    ↓
Use case
    ↓
Business rules
    ↓
Database
```

Tasks should carry all required identifiers and context explicitly.

Never rely on process-local mutable state.

---

## 2.16 Events, outbox, and side effects (optional)

Events are optional. Do not introduce an event subsystem unless multiple consumers, durable delivery, or integration requirements justify it.

For small systems, `transaction.on_commit()` is often enough.

Example:

```python
transaction.on_commit(
    lambda: generer_facture_pdf_task.delay(facture.id)
)
```

But `on_commit()` alone does not provide durable event delivery.

For workflows where losing an event is unacceptable, use an **outbox pattern**.

Conceptually:

```text
transaction
├── update business data
└── insert outbox_event
        ↓
commit
        ↓
outbox dispatcher
        ↓
Celery / external system / integration
```

The outbox event should contain enough information to safely retry processing.

### Event rules

- Events are facts, not commands.
- Events should have stable names.
- Event consumers must tolerate retries.
- Consumers should be idempotent.
- Do not hide important synchronous invariants behind asynchronous events.

For example, stock validation required to complete an order should happen in the order transaction. A notification about the completed order may happen asynchronously.

---

## 2.17 Signals and save()

Signals are allowed only for narrow technical concerns.

Good candidates:

- technical cache invalidation where explicitly designed;
- low-risk integration hooks;
- framework-required behavior.

Bad candidates:

- stock movements;
- accounting entries;
- payment allocation;
- invoice validation;
- automatic document transitions;
- sending email;
- external API calls.

The business workflow should be explicit:

```text
ValiderFacture
    ↓
Facture
    ↓
Comptabilité
    ↓
Notification
```

not hidden across several signal handlers.

### `save()` rule

`save()` must not hide a multi-step business process.

A caller should be able to understand what changing an entity does by reading the calling use case.

---

## 2.18 Document numbering and idempotency

Document numbers must be generated server-side.

Never trust:

```json
{
  "numero_facture": "FAC-2026-0001"
}
```

from the client when the server owns legal numbering.

### Counter pattern

Use a counter table with a unique logical key, for example:

```text
(document_type, period, optional business scope)
```

Inside a transaction:

```text
lock counter row
 ↓
read current number
 ↓
increment counter
 ↓
create document
 ↓
commit
```

### Gaps

Do not promise “no gaps” unless the domain actually requires it and the implementation can guarantee it.

A database sequence may create gaps because of rollbacks or concurrent allocation.

If legal numbering has stricter requirements, implement a transactionally controlled numbering strategy and test it under concurrency.

### Idempotency keys

Use idempotency for operations where retrying the request must not create duplicate business effects.

Typical operations:

- payment creation;
- external webhook processing;
- imports;
- stock movements;
- document generation requests;
- external synchronization.

A typical pattern is:

```text
idempotency_key
+
operation_type
+
unique constraint
```

with the result stored for safe replay when appropriate.

---

## 2.19 Files and external integrations

### Files

Production file storage SHOULD use object storage rather than the application container filesystem.

Rules:

- validate size;
- validate MIME/type;
- never trust user-controlled paths;
- generate safe object names;
- authorize every download;
- scope access according to the owning business object;
- define retention/deletion rules;
- prefer signed URLs for suitable object-storage workflows.

Do not store unbounded files in PostgreSQL merely because it is convenient.

### External APIs

Every external integration must have:

- explicit timeout;
- retry policy;
- idempotency strategy;
- authentication strategy;
- validation of remote responses;
- structured logging without secrets;
- failure classification;
- monitoring.

Never use infinite retries.

Never retry an unsafe operation blindly.

For payment or financial APIs, understand whether the provider's operation is idempotent before implementing retries.

---

## 2.20 Caching

Caching is an optimization, not the primary source of truth.

Rules:

- PostgreSQL remains authoritative.
- Cache keys must be deterministic.
- Cache invalidation strategy must be explicit.
- TTLs must be justified.
- Do not cache highly mutable financial state without a clear consistency model.

Good candidates:

- rarely changing reference data;
- permissions/role metadata;
- expensive read-only reports with acceptable staleness;
- configuration.

Be cautious with:

- stock availability;
- balances;
- payment state;
- financial totals.

### Cache stampede

For expensive cacheable data, consider:

- request coalescing;
- short randomized TTL jitter;
- locking;
- stale-while-revalidate patterns.

Do not add Redis merely because “enterprise systems use Redis”.

---

## 2.21 Observability and auditability

Three concepts must remain distinct.

### Technical logs

Examples:

- request failed;
- database timeout;
- worker retry;
- external API failure.

### Business audit

Examples:

- invoice validated;
- payment cancelled;
- user permission changed;
- financial document approved.

Audit should contain, where appropriate:

- actor;
- action;
- object type;
- object ID;
- timestamp;
- relevant context;
- old/new values for sensitive changes;
- request correlation information.

### Accounting history

Accounting history is not a generic application log.

Journal entries and financial records have their own domain semantics and retention requirements.

### Correlation ID

Every request must have a correlation ID.

It should propagate to:

```text
HTTP request
 ↓
application logs
 ↓
Celery task
 ↓
external call
```

Monitor at minimum:

- request latency;
- error rate;
- database health;
- slow queries;
- worker queue depth;
- failed jobs;
- retry counts;
- external integration failures;
- authentication failures;
- unusual authorization failures.

---

## 2.22 Testing

Testing should reflect architectural risk.

### Unit tests

Use for:

- pure calculations;
- business rules;
- value objects;
- small deterministic transformations.

They should be fast.

### Integration tests

Use real PostgreSQL for:

- ORM behavior;
- constraints;
- transactions;
- locking;
- migrations;
- generated columns;
- database-specific behavior.

Do not mock the database when the purpose of the test is to prove database behavior.

### API tests

Cover:

- success;
- validation failure;
- authentication;
- authorization;
- not found;
- business conflicts;
- response schema;
- pagination;
- filtering;
- idempotency where relevant.

### Concurrency tests

Critical workflows should include race-condition tests.

Especially:

- stock;
- payments;
- counters;
- balances;
- document state transitions.

### Test pyramid

Prefer:

```text
        API / system
          /\
         /  \
        /----\
       /      \
      /--------\
     integration
    /------------\
        unit
```

Most business logic should be testable without the full HTTP stack.

But critical persistence guarantees must be tested against real PostgreSQL.

---

## 2.23 API versioning and compatibility

Treat the API as a contract.

Do not change response fields, meanings, or status codes casually.

Breaking changes should use an explicit versioning strategy.

Possible patterns include:

```text
/api/v1/...
/api/v2/...
```

or another documented project-wide policy.

### Compatibility rules

A change is not automatically “small” because the Python code is small.

Consider whether it changes:

- request schema;
- response schema;
- error code;
- status code;
- pagination behavior;
- ordering guarantees;
- identifier formats;
- date/time semantics.

OpenAPI documentation must reflect the real implementation.

---

## 2.24 Data migrations and deployment safety

Schema changes must be deployable safely.

For production systems, avoid migrations that require prolonged application downtime.

Prefer expand-and-contract patterns for dangerous changes.

### Example

Instead of:

```text
rename column immediately
```

use:

```text
1. Add new column
2. Deploy code that writes both
3. Backfill data
4. Switch reads
5. Stop writing old column
6. Remove old column later
```

### Migration rules

- Every model change has a migration.
- Migrations are committed with the code change.
- Migrations are tested in CI.
- Destructive migrations are explicitly reviewed.
- Production rollback strategy is considered before deployment.
- Large backfills should not block application requests.

### CI migration checks

CI should verify at minimum:

- migrations apply from a clean database;
- migrations apply from a representative previous state where practical;
- tests pass;
- lint/type checks pass;
- security checks pass.

---

## 2.25 AI agent rules and borders

AI coding agents MUST obey the architecture rather than inventing a competing design.

### Before changing code, the agent MUST

1. Identify the bounded context.
2. Identify the business operation being changed.
3. Read the existing module structure.
4. Locate related use cases, rules, exceptions, and public interfaces.
5. Inspect existing authorization logic.
6. Inspect relevant models and database constraints.
7. Inspect related tests.
8. Check whether the change affects API contracts or migrations.
9. Check whether concurrency or idempotency matters.

### During implementation, the agent MUST

- preserve module boundaries;
- use existing architecture patterns;
- use French business vocabulary in code identifiers where the project standard requires it;
- use English developer documentation/comments when required by project convention;
- keep views thin;
- keep serializers focused on API representation and validation;
- choose `ModelSerializer` for simple model-backed CRUD and plain `Serializer` for business commands or composite API contracts;
- use Django models as entities and the Django ORM as the default persistence layer;
- introduce repositories, DAOs, or extra entity layers only when a concrete requirement justifies them;
- place meaningful business rules in the appropriate rules module;
- use typed business exceptions for meaningful business conflicts;
- use transactions for atomic workflows;
- add database constraints when the invariant belongs at the database layer;
- add migrations for model changes;
- update tests;
- update OpenAPI documentation;
- preserve backward compatibility unless a breaking change is explicitly requested;
- avoid unnecessary abstractions.

### The agent MUST NOT

- introduce a competing architecture without explicit approval;
- create generic `Service`, `Manager`, `Helper`, or `Utility` classes without concrete justification;
- create repository/DAO interfaces or separate entity classes without a concrete requirement;
- create a class for every function;
- put major business workflows in serializers;
- put critical workflows in signals;
- hide business workflows in `save()`;
- build inconsistent response formats;
- trust client-provided authorization state;
- bypass another module's public business operation for cross-module writes;
- expose internal persistence objects as API contracts by accident;
- log secrets;
- silently delete historical business records;
- add Redis, Celery, search infrastructure, or microservices without a demonstrated requirement;
- add indexes without understanding the query pattern;
- perform external calls inside critical database transactions unless required and explicitly justified;
- ignore concurrency because the current user count is small;
- treat passing unit tests as proof of production readiness.

### Agent architectural decision rule

Before adding an abstraction, the agent must identify the concrete problem being solved.

The explanation should answer:

```text
What problem exists?
Why does the current architecture fail?
Why is this abstraction the smallest useful solution?
What future complexity does it introduce?
```

If no concrete answer exists, do not add the abstraction.

---

# Chapter 3 — Strict Quality Checklist

Every feature, bugfix, refactor, migration, or API change must pass every applicable item below.

A feature is not complete merely because the tests pass.

## A. Architecture and boundaries

- [ ] The module follows the Django-native modular-monolith architecture defined at the beginning of this guide.
- [ ] Django models remain the entities/persistence representation.
- [ ] No unnecessary entity, repository, DAO, or data-access abstraction was introduced.

- [ ] Change belongs to the correct bounded context.
- [ ] Module ownership remains clear.
- [ ] No circular dependency introduced.
- [ ] No unnecessary abstraction added.
- [ ] Cross-module writes use the owning module's public operation.
- [ ] No project-wide `services/`, `models/`, `views/`, or `utils/` dumping ground introduced.
- [ ] Dependency direction remains understandable.
- [ ] Business logic is not hidden in infrastructure code.

## B. Use case and CRUD structure
- [ ] Simple models use the `crud/` pattern when appropriate.
- [ ] Simple models do not have unnecessary create/update/delete use-case folders.
- [ ] Standard DRF CRUD is used only when there is no meaningful business workflow.
- [ ] Business workflows use dedicated use-case folders.

- [ ] One folder represents one meaningful business operation.
- [ ] Use-case names describe business intent.
- [ ] Files use valid Python `snake_case` names.
- [ ] Empty ceremonial files were not added.
- [ ] No generic service dumping ground created.
- [ ] Simple reads remain simple.
- [ ] Complex reads have a justified query/read service.

## C. Business rules

- [ ] Important business decisions are explicit.
- [ ] Reusable invariants are located in the appropriate rules module.
- [ ] Rules do not perform hidden external side effects.
- [ ] Database queries are not hidden inside pure calculations.
- [ ] Business rules are not duplicated across several views.
- [ ] State transitions are explicit.

## D. Exceptions

- [ ] Input validation uses the normal validation mechanism.
- [ ] Meaningful business conflicts use stable business error codes.
- [ ] No unnecessary exception class was created for trivial validation.
- [ ] Global exception handling is used consistently.
- [ ] Internal implementation details are not exposed.

## E. API response contract

- [ ] Normal JSON responses use the standard envelope.
- [ ] Binary/streaming responses correctly bypass the JSON envelope where necessary.
- [ ] `correlation_id` is present on JSON responses.
- [ ] HTTP status codes are correct.
- [ ] Error codes are stable.
- [ ] OpenAPI matches the implementation.

## F. Serializers

- [ ] Serializer responsibility is limited to API input/output representation and validation.
- [ ] `ModelSerializer` is used deliberately for simple model-backed CRUD only.
- [ ] Plain `Serializer` is used when the payload represents a business command, composite input, or intentionally persistence-independent API contract.
- [ ] A `ModelSerializer` has not been turned into a complex business workflow.
- [ ] Business state transitions are handled by a use case/business operation, not by serialization logic.
- [ ] No hidden business workflow exists in the serializer.
- [ ] Validation is limited to input/API concerns plus appropriate framework-level relation validation.
- [ ] Business-purpose queries are not hidden inside `validate_*()`, `SerializerMethodField`, or representation helpers.
- [ ] No N+1 query was introduced by serialization.
- [ ] No external API calls occur in serialization.
- [ ] No email/PDF generation or multi-aggregate mutation occurs in serialization.
- [ ] Read serialization operates on appropriately fetched data.
- [ ] Serializer design does not create a needless repository/DAO abstraction around Django ORM.

## G. Views

- [ ] View handles HTTP concerns only.
- [ ] Authentication and authorization are enforced.
- [ ] Business workflows are delegated to use cases.
- [ ] No email/PDF/external integration is hidden in the view.
- [ ] No unbounded query exists.
- [ ] Response contract is consistent.

## H. Authorization

- [ ] Authentication is required where appropriate.
- [ ] Permissions are checked server-side.
- [ ] Object-level authorization is applied where necessary.
- [ ] Client-supplied role/permission/user identifiers are not trusted.
- [ ] Authorization behavior is covered by tests.

## I. OOP vs functions

- [ ] Every new class represents a real concept or framework requirement.
- [ ] No class wraps one trivial function.
- [ ] No generic helper/service class was introduced.
- [ ] Inheritance is justified by genuine polymorphism.
- [ ] Pure calculations use functions where clearer.

## J. Models and persistence

- [ ] Models contain appropriate persistence concerns.
- [ ] No hidden multi-step workflow exists in `save()`.
- [ ] Database constraints protect critical invariants.
- [ ] Foreign keys are used where appropriate.
- [ ] Monetary fields use `Decimal` / `NUMERIC`.
- [ ] Historical business records are not casually deleted.
- [ ] `date_creation` and `date_modification` are present where required.

## K. Transactions and concurrency

- [ ] Atomic business operations use transactions.
- [ ] Locking is applied where concurrent updates can violate invariants.
- [ ] Lock ordering is consistent when several rows are locked.
- [ ] Slow external calls are outside the critical transaction where practical.
- [ ] Race conditions have been considered explicitly.
- [ ] Concurrency tests exist for high-risk operations.

## L. Database integrity

- [ ] Unique constraints protect uniqueness.
- [ ] Check constraints protect simple invariants.
- [ ] Foreign keys protect relationships.
- [ ] PostgreSQL-specific behavior is tested on PostgreSQL.
- [ ] Application checks are not the only protection for critical concurrent invariants.

## M. Query performance

- [ ] No N+1 queries.
- [ ] `select_related()` / `prefetch_related()` used appropriately.
- [ ] Collections are paginated.
- [ ] No unbounded `.all()` in request paths.
- [ ] Aggregations are performed efficiently.
- [ ] Query plans have been considered for expensive queries.
- [ ] Indexes correspond to actual access patterns.

## N. Pagination and search

- [ ] Maximum result size is bounded.
- [ ] Filter fields are explicitly supported.
- [ ] Sort fields are explicitly supported.
- [ ] Search queries are bounded and safe.
- [ ] Large search requirements use a dedicated search technology only when justified.

## O. Configuration

- [ ] No environment variable is read directly from business code.
- [ ] `.env.example` is updated.
- [ ] Required production configuration fails fast when missing.
- [ ] No secret is committed.
- [ ] `DEBUG=False` in production.
- [ ] Financial integrity principles from Chapter 1 are treated as architectural constraints for financial code.
- [ ] Inventory integrity principles from Chapter 1 are treated as architectural constraints for stock code.
- [ ] Multi-process and deployment assumptions from Chapter 1 are respected by all stateful code.

## P. Security

- [ ] Authentication enforced where required.
- [ ] Authorization enforced server-side.
- [ ] CSRF is correctly configured for cookie-based authentication.
- [ ] CORS is explicitly configured.
- [ ] Sensitive endpoints are rate-limited where necessary.
- [ ] Passwords use Django's password hashing.
- [ ] Secrets are never logged.
- [ ] Uploaded files are validated.
- [ ] Dependencies are checked for vulnerabilities.
- [ ] Error responses do not expose internals.

## Q. Financial integrity

- [ ] All money uses `Decimal` / `NUMERIC`.
- [ ] Currency rules are explicit.
- [ ] Rounding rules are explicit.
- [ ] Finalized financial documents are not silently mutated.
- [ ] Corrections use the correct reversal/adjustment workflow.
- [ ] Posted accounting entries are immutable where applicable.
- [ ] Ledger history is auditable.
- [ ] Financial operations are atomic.
- [ ] Duplicate financial effects are prevented.

## R. Inventory integrity

- [ ] Stock movements are recorded explicitly.
- [ ] Historical movements are not silently rewritten.
- [ ] Concurrent stock changes are protected.
- [ ] Reservations are distinguished from physical stock where required.
- [ ] Available quantity semantics are explicit.
- [ ] Retryable operations are idempotent.
- [ ] Stock-related race conditions are tested.

## S. Document numbering and idempotency

- [ ] Client cannot choose authoritative legal document numbers.
- [ ] Number generation is concurrency-safe.
- [ ] Numbering scope is explicitly defined.
- [ ] Idempotency keys are used where retries could duplicate business effects.
- [ ] Duplicate webhook/import/payment processing is prevented.

## T. Background jobs

- [ ] Long-running work is asynchronous where appropriate.
- [ ] Tasks are idempotent where possible.
- [ ] Retry policies are explicit.
- [ ] Timeouts are explicit.
- [ ] Tasks invoke use cases rather than duplicating workflows.
- [ ] Task failures are observable.
- [ ] Post-commit enqueueing is used where necessary.

## U. Signals and save()

- [ ] No critical business workflow is hidden in signals.
- [ ] No critical business workflow is hidden in `save()`.
- [ ] Side effects are explicit and traceable.
- [ ] Signal usage is documented and justified.

## V. Files and integrations

- [ ] File size is limited.
- [ ] File type is validated.
- [ ] Client filenames/paths are not trusted.
- [ ] File access is authorized.
- [ ] External APIs have timeouts.
- [ ] Retry safety has been evaluated.
- [ ] External failures are observable.
- [ ] Credentials are centrally configured.

## W. Caching

- [ ] Cache is not treated as source of truth.
- [ ] Cache keys are deterministic.
- [ ] TTL and invalidation behavior are understood.
- [ ] Financially sensitive mutable data is not cached without a clear consistency strategy.
- [ ] Cache stampede risk is considered for expensive values.

## X. Observability and audit

- [ ] Errors are logged with sufficient context.
- [ ] Correlation ID is propagated.
- [ ] Important business actions are auditable.
- [ ] Technical logs and business audit are separated conceptually.
- [ ] Accounting history is not confused with application logs.
- [ ] No sensitive data is logged.
- [ ] Metrics exist for critical operations.

## Y. Testing

- [ ] Unit tests cover business calculations/rules.
- [ ] Integration tests use PostgreSQL for database behavior.
- [ ] API tests cover success and failure paths.
- [ ] Authorization is tested.
- [ ] Not-found and conflict cases are tested.
- [ ] Database constraints are tested where important.
- [ ] Concurrency is tested for high-risk workflows.
- [ ] Idempotency is tested where applicable.
- [ ] Migrations apply cleanly.

## Z. API compatibility

- [ ] Request schema is documented.
- [ ] Response schema is documented.
- [ ] Error codes are documented.
- [ ] Pagination behavior is documented.
- [ ] Breaking changes use the project's versioning strategy.
- [ ] Existing consumers have not been broken accidentally.

## AA. Migrations and deployment

- [ ] Model changes include migrations.
- [ ] Migrations pass CI.
- [ ] Destructive changes are reviewed.
- [ ] Large backfills do not unnecessarily block requests.
- [ ] Expand-and-contract is used for risky schema changes.
- [ ] Deployment order is safe.
- [ ] Rollback implications are understood.
- [ ] Health and readiness checks are correct.

## AB. Code quality

- [ ] Names are explicit and consistent.
- [ ] Public functions have type hints where appropriate.
- [ ] No dead code was introduced or retained unnecessarily.
- [ ] No broad `except Exception` hides failures.
- [ ] No unnecessary dependency was introduced.
- [ ] No unnecessary framework or infrastructure was introduced.
- [ ] Linting passes.
- [ ] Type checks pass where adopted.
- [ ] Documentation reflects meaningful architectural decisions.

## AC. Final architectural review

Before considering the change production-ready, answer all of these:

- [ ] Is the code in the correct bounded context?
- [ ] Does this use case have a clear business purpose?
- [ ] Is the business rule explicit rather than hidden?
- [ ] Is a class genuinely necessary?
- [ ] Does the API follow the established contract?
- [ ] Can another module bypass the owning module's critical rules?
- [ ] Can two concurrent requests corrupt state?
- [ ] Can a retry produce a duplicate effect?
- [ ] Can a failure leave partial business state?
- [ ] Are database constraints strong enough?
- [ ] Could this endpoint unexpectedly issue hundreds of queries?
- [ ] Are heavy operations asynchronous where appropriate?
- [ ] Are external calls safely isolated from transactions?
- [ ] Is the operation observable?
- [ ] Is the operation auditable where required?
- [ ] Can historical financial/inventory state be reconstructed?
- [ ] Is the migration safe in production?
- [ ] Does this change add complexity without solving a concrete problem?
- [ ] Would a new developer understand the implementation without tribal knowledge?

A change that fails any security, data-integrity, transaction, concurrency, authorization, migration-safety, or API-contract requirement MUST NOT be considered production-ready.

---

# Architectural Summary

The target architecture can be reduced to one principle:

```text
                    ERP
                     │
              Modular Monolith
                     │
      ┌──────────────┼──────────────┐
      ↓              ↓              ↓
    Ventes         Stocks      Comptabilité
      │              │              │
      └──────────────┼──────────────┘
                     ↓
                Use Cases
                     ↓
             Business Rules
                     ↓
               Django ORM
                     ↓
                PostgreSQL
```

For reads:

```text
HTTP
 ↓
View
 ↓
Query / Read Service
 ↓
PostgreSQL
 ↓
Serializer
 ↓
API response
```

The serializer is the API representation layer. It does not need to own persistence access.

For simple model-backed CRUD, the read/write flow may use `ModelSerializer` directly with DRF generic views or viewsets:

```text
HTTP
 ↓
ViewSet
 ↓
ModelSerializer
 ↓
Django ORM
 ↓
PostgreSQL
```

For more complex operations, keep the API schema separate from the business workflow:

```text
HTTP
 ↓
Input Serializer
 ↓
Use Case
 ↓
Django Model / ORM
 ↓
PostgreSQL

Django Model / QuerySet
 ↓
Output Serializer
 ↓
API response
```

For writes:

```text
HTTP
 ↓
View
 ↓
Serializer validation
 ↓
Use Case
 ├── Authorization
 ├── Load state
 ├── Business rules
 ├── Transaction
 ├── Database constraints
 ├── Mutation
 └── Post-commit side effects
 ↓
PostgreSQL
```

For asynchronous work:

```text
Committed business operation
        ↓
Outbox / on_commit
        ↓
Queue
        ↓
Worker
        ↓
Use Case
```

The architecture is deliberately conservative about abstraction. It should become **more modular before it becomes more distributed**.

The ERP should first scale by:

1. keeping business boundaries clean;
2. keeping PostgreSQL integrity strong;
3. making transactions correct;
4. preventing N+1 and unbounded queries;
5. making critical operations idempotent;
6. moving heavy work to workers;
7. adding caching only where justified;
8. adding read replicas for reporting when needed;
9. partitioning large append-only tables when measurements justify it;
10. extracting a service only when a real operational or organizational boundary requires it.

This is the default architecture standard. Deviations should be explicit, documented, and justified by a concrete engineering requirement.
