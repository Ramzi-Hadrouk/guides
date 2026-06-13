# Django-Ninja Architecture Guide (Scalable Enterprise Version)

This architecture is designed for:

* Large SaaS systems
* Multi-team development
* Django + Django-Ninja applications
* Any SQL database (PostgreSQL, MySQL, SQLite, …) + Redis + Celery + Django Channels
* Long-term maintainability
* Feature isolation
* Domain-driven backend systems

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
│   ├── utilisateurs/               # Module: users & auth
│   ├── autorisations/              # Module: RBAC (roles, memberships, permissions)
│   │
│   │   # ─── Domain Group: Clinical ─────────────────────────────────────────
│   ├── medecins/                   # Module: doctors
│   ├── cliniques/                  # Module: clinics + self-registration
│   ├── patients/                   # Module: patient records
│   │
│   │   # ─── Domain Group: Scheduling & Booking ─────────────────────────────
│   ├── rendez_vous/                # Module: appointments (booking, OCC)
│   ├── scheduling/                 # Module: time slots & availability
│   ├── sous_services/              # Module: bookable subservices
│   │
│   │   # ─── Domain Group: Discovery ─────────────────────────────────────────
│   ├── recherche/                  # Module: full-text patient-facing search (read-only)
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
│       │       ├── creer_<entity>_service.py
│       │       ├── modifier_<entity>_service.py
│       │       └── supprimer_<entity>_service.py
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

* Keep related bounded contexts visible in the codebase
* Separate clinical, scheduling, identity, and discovery concerns clearly
* Make ownership clearer for teams
* Improve maintainability as the project grows
* Reduce cognitive load by making bounded contexts explicit

**Standard domain groups:**

| Domain Group | Modules | Responsibilities |
| --- | --- | --- |
| Identity & Access | utilisateurs, autorisations | Auth, users, RBAC, roles, memberships |
| Clinical | medecins, cliniques, patients | Doctor records, clinic management, patient records |
| Scheduling & Booking | rendez_vous, scheduling, sous_services | Appointments, time slots, bookable subservices |
| Discovery | recherche | Full-text search, public browsing |
| Integrations | integrations/* | Third-party system adapters (e.g. Medex) |

When adding a new module, assign it to an existing domain group, or establish a new group if it represents a genuinely independent bounded context with its own entities, writes, and reads.

---

# Layer Responsibilities

## Infrastructure & Configuration Layers

| Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| config/settings/ | Environment-specific Django configuration. | Installed apps, database config, middleware, Celery broker, Redis URL, CORS, JWT settings. | Business logic, feature flags, domain rules. |
| config/api.py | Django-Ninja root API instance and global router registration. | `NinjaAPI` instance, global exception handlers, `api.add_router()` calls. | Business logic, direct endpoint definitions. |
| config/celery.py | Celery application instance and task auto-discovery. | `Celery()` instance, `autodiscover_tasks()`. | Task logic, service calls. |
| core/ | Infrastructure layer providing the system's technical, business-agnostic foundation. | Base exception hierarchy, RBAC resolver, permission decorator, pagination helpers. | Business-specific models, feature execution, domain rules. |
| core/exceptions.py | Base error hierarchy that all domain exceptions extend. | `ApplicationError`, `NotFoundError`, `ConflictError`, error-to-HTTP status mapping. | Feature-specific exception subclasses (those live in each module's `domain/exceptions.py`). |
| core/permissions.py | Single source of truth for all RBAC resolution. | `requires_permission()` decorator, `a_la_permission()` resolver, live DB permission lookup. | Business logic beyond permission checking, ORM queries for non-auth purposes. |

---

## Module-Level Layers

| Layer | Description & Responsibilities | Allowed / Contains | Forbidden |
| --- | --- | --- | --- |
| api/router.py | HTTP controller. For reads, imports and calls the repository directly. For writes, imports and calls the service. | `@router` endpoints, schema validation, permission decorators, direct imports of repository and service classes. | Business logic, domain rules, ORM model imports, transaction management outside of service calls. |
| api/schemas.py | Pydantic input/output shapes for the HTTP boundary. | Request schemas, response schemas, field-level syntax validators. | Domain entities, ORM models, business constraint validation. |
| domain/entities.py | Pure, framework-independent domain models. | Pydantic `BaseModel` domain objects, computed properties, domain-level constraints. | ORM imports, Django imports, Redis, Celery, any I/O. |
| domain/value_objects.py | Immutable, self-validating types wrapping primitives. | `Email`, `PhoneNumber`, `ClinicSlug` and similar types with built-in validators. | Mutable state, ORM references. |
| domain/exceptions.py | Domain-specific exception classes. | `MedecinIntrouvableError`, `SlotIndisponibleError` — all subclasses of `core.exceptions.ApplicationError`. | HTTP status codes, Django exceptions. |
| domain/rules.py | Pure business rule functions and strategy abstractions. | `peut_annuler_rdv()`, `calculer_priorite()`, `PrioriteStrategie` ABC and its subclasses. | I/O operations, ORM access, service calls. |
| services/ | Write orchestration layer. One class per use-case, one method: `executer()`. | Domain imports, repository imports, cache imports, `transaction.atomic()`, post-commit event broadcasting. | `api/`, HTTP objects, ORM model imports (`models.py`). |
| repositories/django_repo.py | Concrete ORM data-access implementation. Owns all reads and writes for this module. | Django ORM queries, `_map_to_domain()` mappers, `_map_to_orm()` converters. Read methods and write methods are clearly sectioned. | Business logic, validation, HTTP concerns, service calls. |
| cache/redis_repo.py | Redis interaction layer. Present only in modules that maintain live real-time state. | Redis read/write operations, key construction, TTL management, Lua scripts for atomic operations. | Services, repositories, API layer. |
| tasks/ | Thin Celery task wrappers. Max 10 lines per task file. | `@shared_task` definitions, service class import and `executer()` call. | Business logic, ORM queries, multi-step orchestration. |
| consumers.py | Django Channels WebSocket consumers. | Subscribe, receive, forward to client. Zero business logic. | Service calls, ORM queries, domain rules. |
| models.py | ORM model definition. | Field definitions, `Meta` class, `__str__`. Nothing else. | Business methods, validation logic, queryset filtering, `save()` overrides with logic. |
| tests/unit/ | Fast, isolated unit tests with no DB or network I/O. | Domain entity tests, business rule tests, pure function tests, service tests with mocked repos. | Live ORM, HTTP calls, real Redis. |
| tests/integration/ | Full-stack tests against the real DB. | Router endpoint tests, service-to-DB flow tests. | Production Redis, external third-party calls. |

---

# SOLID Principles — Enforced Patterns

## Single Responsibility

One service class per use-case, one public method: `executer()`.

```python
# ❌ WRONG — One class doing multiple unrelated things
class CliniqueService:
    def creer_clinique(self, donnees): ...
    def modifier_horaires(self, donnees): ...
    def supprimer_clinique(self, id): ...


# ✅ CORRECT — One class per file in a domain-specific folder

# cliniques/services/clinique_services/creer_clinique_service.py
class CreerCliniqueService:
    def executer(self, donnees: dict) -> Clinique: ...

# cliniques/services/clinique_services/modifier_horaires_service.py
class ModifierHorairesService:
    def executer(self, clinique_id: UUID, donnees: dict) -> Clinique: ...

# cliniques/services/clinique_services/supprimer_clinique_service.py
class SupprimerCliniqueService:
    def executer(self, clinique_id: UUID) -> None: ...
```

**Service directory example — `autorisations` module:**

```
apps/autorisations/services/
├── __init__.py
├── membership_services/
│   ├── __init__.py
│   ├── inviter_membre_service.py
│   ├── desactiver_membre_service.py
│   └── changer_role_service.py
└── role_services/
    ├── __init__.py
    ├── creer_role_service.py
    └── mettre_a_jour_permissions_service.py
```

---

## Open/Closed — Strategy Pattern

Variant behaviour is added by subclassing, never by editing existing service code.

```python
# apps/rendez_vous/domain/rules.py

from abc import ABC, abstractmethod


class PrioriteStrategie(ABC):
    @abstractmethod
    def calculer_score(self, ordre: int) -> float: ...


class PrioriteNormale(PrioriteStrategie):
    def calculer_score(self, ordre: int) -> float:
        return float(ordre)


class PrioriteUrgence(PrioriteStrategie):
    def calculer_score(self, ordre: int) -> float:
        return -1000.0 + float(ordre)
```

Services consume the strategy via a parameter — a new priority type requires a new subclass, never an `if/elif` chain in the service.

---

## Liskov Substitution

Repository classes must honour the full contract they advertise. If you swap `DjangoMedecinRepository` for a `MockMedecinRepository` in tests, every method must behave identically in terms of return types and raised exceptions. The mock is a drop-in replacement — no special-casing allowed.

---

## Interface Segregation

Repositories expose only what their consumers actually call. Read-heavy flows (e.g. `recherche` endpoints) only touch read methods; they never import write operations. If a module's repository grows unwieldy, split it into `django_read_repo.py` and `django_write_repo.py` without changing the callers.

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
# apps/medecins/services/medecin_services/creer_medecin_service.py

from django.db import transaction

from apps.medecins.repositories.django_repo import DjangoMedecinRepository
from apps.medecins.domain.entities import Medecin
from apps.medecins.domain.exceptions import MedecinExisteDejaError


class CreerMedecinService:
    def executer(self, donnees: dict) -> Medecin:
        repo = DjangoMedecinRepository()

        if repo.obtenir_par_email(donnees["email"]):
            raise MedecinExisteDejaError()

        with transaction.atomic():
            return repo.sauvegarder(donnees)
```

**Real-world examples across modules:**

```python
# apps/utilisateurs/services/auth_services/login_service.py
class LoginService:
    def executer(self, email: str, password: str) -> TokenPair: ...

# apps/cliniques/services/clinique_services/enregistrer_clinique_service.py
class EnregistrerCliniqueService:
    """
    Clinic self-registration — one atomic transaction:
    1. Create clinic record (with owner_user_id)
    2. Seed clinic_roles (ADMIN, DOCTOR, STAFF)
    3. Seed role_permissions per default matrix
    4. Create clinic_memberships for owner → ADMIN role
    """
    def executer(self, donnees: dict) -> Clinique: ...

# apps/autorisations/services/membership_services/inviter_membre_service.py
class InviterMembreService:
    def executer(self, donnees: dict) -> dict: ...
```

**Cross-module reads inside a write service** — import the other module's repository directly:

```python
# apps/rendez_vous/services/rendez_vous_services/creer_rendez_vous_service.py

from django.db import transaction

from apps.rendez_vous.repositories.django_repo import DjangoRendezVousRepository
from apps.medecins.repositories.django_repo import DjangoMedecinRepository   # cross-module read
from apps.rendez_vous.domain.exceptions import MedecinIntrouvableError, SlotIndisponibleError


class CreerRendezVousService:
    def executer(self, donnees: dict) -> RendezVous:
        medecin_repo = DjangoMedecinRepository()
        if not medecin_repo.obtenir_par_id(donnees["medecin_id"]):
            raise MedecinIntrouvableError()

        repo = DjangoRendezVousRepository()
        with transaction.atomic():
            return repo.sauvegarder(donnees)
```

---

# Repository Layer — Data Access

The repository is the **only place that touches the ORM**. It owns all read and write queries for its module and maps between ORM models and domain entities. Read methods and write methods are clearly sectioned inside the same class.

```python
# apps/medecins/repositories/django_repo.py

from uuid import UUID

from apps.medecins.models import MedecinORM
from apps.medecins.domain.entities import Medecin


class DjangoMedecinRepository:

    # ── Reads ─────────────────────────────────────────────────────────────────

    def obtenir_par_id(self, medecin_id: UUID) -> Medecin | None:
        try:
            return self._map_to_domain(MedecinORM.objects.get(id=medecin_id))
        except MedecinORM.DoesNotExist:
            return None

    def obtenir_par_email(self, email: str) -> Medecin | None:
        try:
            return self._map_to_domain(MedecinORM.objects.get(email=email.lower()))
        except MedecinORM.DoesNotExist:
            return None

    def lister_par_clinique(self, clinic_id: UUID) -> list[Medecin]:
        return [
            self._map_to_domain(orm)
            for orm in MedecinORM.objects.filter(clinic_id=clinic_id, est_actif=True)
        ]

    # ── Writes ────────────────────────────────────────────────────────────────

    def sauvegarder(self, donnees: dict) -> Medecin:
        orm = MedecinORM.objects.create(**self._map_to_orm(donnees))
        return self._map_to_domain(orm)

    def mettre_a_jour(self, medecin_id: UUID, donnees: dict) -> Medecin:
        MedecinORM.objects.filter(id=medecin_id).update(**self._map_to_orm(donnees))
        return self._map_to_domain(MedecinORM.objects.get(id=medecin_id))

    def supprimer(self, medecin_id: UUID) -> None:
        MedecinORM.objects.filter(id=medecin_id).delete()

    # ── Mappers ───────────────────────────────────────────────────────────────

    def _map_to_domain(self, orm: MedecinORM) -> Medecin:
        return Medecin(
            id=orm.id,
            email=orm.email,
            nom=orm.nom,
            prenom=orm.prenom,
            est_actif=orm.est_actif,
        )

    def _map_to_orm(self, donnees: dict) -> dict:
        return {
            "email": donnees["email"].lower(),
            "nom": donnees["nom"],
            "prenom": donnees["prenom"],
        }
```

---

# API Layer — HTTP Controllers

The API layer is **thin by law**. It delegates immediately — never implements. For reads (GET), it imports and calls the repository. For writes (POST/PUT/PATCH/DELETE), it imports and calls the service.

```python
# apps/medecins/api/router.py

from uuid import UUID
from ninja import Router
from ninja.errors import HttpError

from core.permissions import requires_permission
from apps.medecins.api.schemas import CreerMedecinSchema, MedecinSchema
from apps.medecins.repositories.django_repo import DjangoMedecinRepository
from apps.medecins.services.medecin_services.creer_medecin_service import CreerMedecinService
from apps.medecins.services.medecin_services.supprimer_medecin_service import SupprimerMedecinService

router = Router()


# ── GET ───────────────────────────────────────────────────────────────────────

@router.get("/clinics/{clinic_id}/medecins/", response=list[MedecinSchema])
def lister_medecins(request, clinic_id: UUID):
    repo = DjangoMedecinRepository()
    return repo.lister_par_clinique(clinic_id)


@router.get("/medecins/{medecin_id}/", response=MedecinSchema)
def obtenir_medecin(request, medecin_id: UUID):
    repo = DjangoMedecinRepository()
    medecin = repo.obtenir_par_id(medecin_id)
    if not medecin:
        raise HttpError(404, "Médecin introuvable")
    return medecin


# ── POST / PUT / DELETE ───────────────────────────────────────────────────────

@router.post("/clinics/{clinic_id}/medecins/", response=MedecinSchema)
@requires_permission("medecins:create")
def creer_medecin(request, clinic_id: UUID, payload: CreerMedecinSchema):
    service = CreerMedecinService()
    return service.executer(payload.dict() | {"clinic_id": clinic_id})


@router.delete("/medecins/{medecin_id}/")
@requires_permission("medecins:delete")
def supprimer_medecin(request, medecin_id: UUID):
    service = SupprimerMedecinService()
    service.executer(medecin_id)
    return {"success": True}
```

---

# RBAC System

> **This section governs `core/permissions.py` and the `autorisations` module.** The permission decorator touches every API endpoint in every module, so the contract here is non-negotiable.

## Architecture Decision: Manual Tenant-Scoped RBAC

The platform uses manual RBAC instead of a third-party library (django-guardian, django-rules, django-role-permissions). The rationale:

| Requirement | Library Support | Manual |
| --- | :---: | :---: |
| Per-clinic role scoping (same user, different roles in different clinics) | ❌ | ✅ |
| Runtime-configurable permissions (clinic admin edits permissions via UI) | ❌ | ✅ |
| Row-level scoping (`appointments:own:read`) | ❌ | ✅ |
| No library deprecation risk | ❌ | ✅ |

---

## `core/permissions.py` — Full Contract

```python
# core/permissions.py

from uuid import UUID
from functools import wraps
from ninja.errors import HttpError

from apps.autorisations.repositories.django_repo import DjangoAutorisationRepository


# ─── Core Resolver ────────────────────────────────────────────────────────────

def obtenir_permissions(user_id: UUID, clinic_id: UUID) -> set[str]:
    """
    Single resolver — everything flows through here.
    Returns the full permission set for a user in a clinic.
    Live DB query — no cache.
    """
    repo = DjangoAutorisationRepository()
    return repo.obtenir_permissions(user_id, clinic_id)


def a_la_permission(user_id: UUID, clinic_id: UUID, permission: str) -> bool:
    return permission in obtenir_permissions(user_id, clinic_id)


# ─── Django-Ninja Decorator ───────────────────────────────────────────────────

def requires_permission(permission: str):
    """
    Decorator for Django-Ninja endpoints.
    Usage: @requires_permission("appointments:write")
    Resolves clinic_id from path kwargs first, then falls back to request attribute.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            clinic_id = kwargs.get("clinic_id") or getattr(request, "clinic_id", None)
            if not clinic_id or not a_la_permission(request.user.id, clinic_id, permission):
                raise HttpError(403, f"Permission requise: {permission}")
            return func(request, *args, **kwargs)
        return wrapper
    return decorator
```

---

## Permission String Convention

All permission strings use explicit `(resource:action)` format. No wildcards. No `manage` shorthand — always expand to discrete verbs.

```
# Clinical
medecins:create       medecins:update       medecins:delete       medecins:read

# Scheduling
appointments:write    appointments:read     appointments:own:read

# Clinic management
clinic:settings:write clinic:settings:read

# RBAC
members:invite        members:deactivate    members:read
roles:create          roles:update          roles:delete
```

---

## Cache Invalidation Contract

Any service that mutates `clinic_memberships` or `role_permissions` **must** call the cache invalidation helper **after** the DB transaction commits. Never inside `transaction.atomic()`.

```python
# apps/autorisations/services/membership_services/inviter_membre_service.py

from django.db import transaction

from apps.autorisations.repositories.django_repo import DjangoAutorisationRepository
from apps.autorisations.cache.redis_repo import AutorisationsCacheRepo


class InviterMembreService:
    def executer(self, donnees: dict) -> dict:
        repo = DjangoAutorisationRepository()
        cache = AutorisationsCacheRepo()

        with transaction.atomic():
            membership = repo.creer_membership(donnees)

        # Cache invalidation AFTER commit — never inside the atomic block
        cache.invalider(clinic_id=donnees["clinic_id"], user_id=donnees["user_id"])
        return membership
```

---

## Row-Level Scoping (`appointments:own:read`)

When a user holds only `appointments:own:read` (not the full `appointments:read`), the router applies an additional filter before calling the repository. Row-level scoping logic lives in the API layer (not in the repository, not in the service).

```python
# apps/rendez_vous/api/router.py

@router.get("/clinics/{clinic_id}/appointments/", response=list[RendezVousSchema])
def lister_rendez_vous(request, clinic_id: UUID):
    perms = obtenir_permissions(request.user.id, clinic_id)
    repo  = DjangoRendezVousRepository()

    if "appointments:read" in perms:
        return repo.lister_par_clinique(clinic_id)

    if "appointments:own:read" in perms:
        auth_repo = DjangoAutorisationRepository()
        doctor_id = auth_repo.obtenir_medecin_par_user(request.user.id, clinic_id)
        return repo.lister_par_medecin(clinic_id, doctor_id) if doctor_id else []

    raise HttpError(403, "Permission requise")
```

---

# Real-Time State Layer (Redis)

> **This section applies only to modules that maintain live, distributed state.** Purely CRUD modules (`medecins`, `cliniques`, `utilisateurs`, `autorisations`) do **not** need a `cache/` folder.

## Deciding If Your Module Needs Redis

| Module needs Redis? | Criteria |
| --- | --- |
| ✅ Yes | Live counters, priority queues, real-time availability, pub/sub, TTL-based lookups |
| ❌ No | Simple CRUD, data that only changes on explicit user action, no real-time consumers |

| Module | Needs Redis? | Reason |
| --- | :---: | --- |
| `rendez_vous` | ❌ | Booking uses OCC on `time_slots` (DB-level); no live counters |
| `scheduling` | ⚠️ | Cache infrastructure exists but is disabled — preserved for future use |
| `recherche` | ✅ | RediSearch index for multi-clinic discovery |
| `autorisations` | ✅ | Permission cache (TTL 300s, key: `perm:{clinic_id}:{user_id}`) |
| `cliniques` | ❌ | Pure CRUD |
| `utilisateurs` | ❌ | Pure CRUD — rate limiting belongs at infrastructure/nginx layer |
| `sous_services` | ❌ | CRUD |

---

## Key Naming Convention

All Redis keys follow `<domain>:<id>:<type>`. Use French segment names.

```
# Queue module
file:{id_file}:compteur                         → STRING  — atomic INCR
file:{id_file}:etat                             → HASH    — live state
file:{id_file}:tickets                          → ZSET    — priority queue

# Slot availability
slots:{clinic_id}:{sous_service_id}:{date}      → HASH    — TTL 60s

# Permission cache
perm:{clinic_id}:{user_id}                      → STRING  — TTL 300s

# Search cache
search:{clinic_id}:{query}:{month}              → STRING  — TTL 3600s

# Generic patterns
<domain>:{id}:etat                              → HASH    — live state snapshot
pubsub:<domain>:{id}                            → CHANNEL — entity-scoped events
session:<domain>:{code}                         → STRING  — short-lived (with TTL)
```

---

## Redis Failure Recovery

* Every Redis write must have a corresponding database write first. The SQL database is the source of truth.
* A Celery management command must exist to rebuild Redis state from the DB on reconnect.
* The rebuild command must be idempotent — safe to run multiple times without side effects.
* Never call `KEYS *` in production — use `SCAN` + `DEL` in batches.

---

# WebSocket + Pub/Sub Rules

## Event Flow (Always in this order)

```
Service → DB write → Redis state update → Redis PUBLISH → Channel consumer → WebSocket push
```

Never skip steps. The DB write always precedes the Redis write.

## Consumer Rules

* Consumers only subscribe and forward. **Zero business logic in consumers.**
* Consumers must handle disconnections gracefully — must never crash the server.
* Channel names are entity-scoped:

```python
"file_globale"          # queue module — all queue events
"medecin_{id}"          # doctor-scoped events
"rendez_vous_{id}"      # appointment-scoped events
"clinique_{id}"         # clinic-scoped events
```

---

# Cross-Module Communication

When a service or the API needs data from another module, it imports that module's `django_repo.py` directly. No injected interfaces, no adapter layers — just a plain import.

```python
# ✅ CORRECT — service reads from another module via direct repository import
# apps/rendez_vous/services/rendez_vous_services/creer_rendez_vous_service.py

from apps.rendez_vous.repositories.django_repo import DjangoRendezVousRepository
from apps.medecins.repositories.django_repo import DjangoMedecinRepository
from apps.autorisations.repositories.django_repo import DjangoAutorisationRepository


class CreerRendezVousService:
    def executer(self, donnees: dict) -> RendezVous:
        if not DjangoMedecinRepository().obtenir_par_id(donnees["medecin_id"]):
            raise MedecinIntrouvableError()

        with transaction.atomic():
            return DjangoRendezVousRepository().sauvegarder(donnees)


# ✅ CORRECT — router assembles a composite read from multiple modules
# apps/recherche/api/router.py

from apps.recherche.repositories.django_repo import DjangoRechercheRepository
from apps.sous_services.repositories.django_repo import DjangoSousServiceRepository


@router.get("/search/")
def rechercher(request, q: str, clinic_id: UUID | None = None):
    repo = DjangoRechercheRepository()
    return repo.rechercher(clinic_id=clinic_id, requete=q)


# ❌ WRONG — importing another module's ORM model directly
from apps.medecins.models import MedecinORM            # forbidden outside medecins/repositories/
from apps.autorisations.models import RoleORM          # forbidden outside autorisations/repositories/
from apps.sous_services.models import SousServiceORM   # forbidden outside sous_services/repositories/
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
forbidden_modules = ["apps.autorisations.repositories.django_repo"]
# All permission resolution must go through core/permissions.py

[[tool.importlinter.contracts]]
name = "recherche module is read-only — no services allowed"
type = "forbidden"
source_modules = ["apps.recherche"]
forbidden_modules = ["apps.recherche.services"]
```

---

# Naming Conventions

| Layer | Language | Rule / Example |
| --- | --- | --- |
| ORM model class names | French | `RendezVousORM`, `MedecinORM`, `CliniqueORM`, `SousServiceORM`, `RoleCliniqueORM` |
| ORM model field names | French | `numero_ticket`, `statut`, `cree_le`, `id_medecin`, `est_actif`, `duree_min`, `vecteur_recherche` |
| Domain entity names | French | `RendezVous`, `Medecin`, `Clinique`, `SousService`, `MembreClinique` |
| Repository class names | English | `DjangoMedecinRepository`, `DjangoAutorisationRepository`, `DjangoRechercheRepository` |
| Repository method names | French | `sauvegarder`, `obtenir_par_id`, `lister_par_clinique`, `supprimer`, `obtenir_permissions` |
| Service class names | English | `CreerMedecinService`, `InviterMembreService`, `EnregistrerCliniqueService` |
| Service method name | French | `executer` — always singular; the class name carries the intent |
| Pydantic schema field names | French | `numero_ticket`, `statut`, `id_medecin`, `id_sous_service`, `duree_min` |
| File naming | snake_case French | `creer_medecin_service.py`, `django_repo.py`, `redis_repo.py` |
| Redis key segments | French | `file:{id}:etat`, `slots:{clinic_id}:{sous_service_id}:{date}`, `perm:{clinic_id}:{user_id}` |
| Permission strings | English | `appointments:write`, `appointments:own:read`, `clinic:settings:write` |
| Python local variable names | French | `rendez_vous`, `medecin`, `clinique`, `sous_service`, `resultats` |
| API endpoint URLs | English | `/api/v1/appointments/`, `/api/v1/clinics/{id}/members/`, `/api/v1/search/` |
| Documentation and comments | English | All in-code docs and PR descriptions stay in English |
| Full-text search vector field *(PostgreSQL only)* | French | `vecteur_recherche` — a `TSVECTOR` computed column; applies only when using PostgreSQL. On other engines use the database's native full-text equivalent and rename accordingly. |

---

# Architectural Rules

* **Rule 1 (Domain Purity):** The `domain/` layer must stay 100% pure: no Django imports, no ORM models, no Redis, no Celery, no HTTP concerns. Test: `pytest apps/<module>/domain/` must pass with `DJANGO_SETTINGS_MODULE` unset.

* **Rule 2 (Thin API Layer):** `router.py` contains zero business logic. For reads: import and call the repository directly. For writes: import and call the service. No orchestration beyond that single delegation.

* **Rule 3 (Services Own All Writes):** Every POST/PUT/PATCH/DELETE must go through a service class with an `executer()` method. Never write to the ORM directly from `router.py`.

* **Rule 4 (Repositories Own ORM Access):** ORM model imports (`models.py`) are strictly forbidden everywhere except `repositories/django_repo.py` and `tasks/`. Any layer that needs data goes through the repository.

* **Rule 5 (One Service, One Use-Case):** A service class has exactly one public method (`executer()`) performing exactly one use-case. If two actions are needed, that is two service classes in two files.

* **Rule 6 (No Constructor Injection):** Services and repositories are never passed as parameters to `__init__`. All dependencies are imported directly at the top of the file and instantiated at call time. No DI containers. No Protocol interfaces.

* **Rule 7 (No Django Signals):** Never use Django signals for domain side effects. Call `core/websocket.py` broadcast helpers explicitly from services, after the DB write, outside `transaction.atomic()`.

* **Rule 8 (Thin Tasks):** Celery tasks are thin shims — they instantiate a service and call `executer()`. Maximum 10 lines per task file. All logic lives in the service.

* **Rule 9 (Zero Consumer Logic):** WebSocket consumers subscribe, receive, and forward. They never call services, touch the ORM, or apply business rules.

* **Rule 10 (Exception Hierarchy):** All domain exceptions extend `core.exceptions.ApplicationError`. Services raise domain exceptions. The API maps them to HTTP status codes via a global exception handler in `config/api.py`.

* **Rule 11 (Redis After DB):** Every Redis write must be preceded by a successful database write. The SQL database is the source of truth. Redis is the speed layer. They are never out of sync for more than one operation.

* **Rule 12 (Cache Invalidation Outside Transactions):** Never call cache invalidation functions inside `transaction.atomic()`. Call them after the `with` block closes, or register them via `transaction.on_commit()`.

* **Rule 13 (No Cross-Module ORM Imports):** When a service or the router needs data from another module, it imports and uses that module's `django_repo.py`. Direct imports of another module's `models.py` are forbidden everywhere except within that module's own repository.

* **Rule 14 (Permission Resolution via Core):** All permission checks must flow through `core/permissions.py`. No service, repository, or router may directly query `clinic_memberships` or `role_permissions` for authorization purposes.

* **Rule 15 (Explicit Permission Strings):** Permission strings are explicit `(resource:action)` tuples. No wildcards. No `manage` shorthand — expand to `create`, `update`, `delete` individually.

* **Rule 16 (System Roles Are Immutable):** System roles (`is_system = TRUE`) may not be deleted or have their permissions modified. Enforce this as a domain rule in `autorisations/domain/rules.py`.

* **Rule 17 (Clinic Must Have One Admin):** A clinic must always have at least one active ADMIN member. `desactiver_membre_service` and `changer_role_service` must verify this invariant before committing.

* **Rule 18 (No Naming Dumping Grounds):** Folder names like `misc/`, `helpers/`, `common/`, or `other/` are strictly banned. Every folder must have a clear, bounded responsibility.

* **Rule 19 (Encapsulate Business Conditionals):** Never write loose boolean conditions inside routers or services (e.g. `user.role == 'ADMIN' and clinic.is_active`). Wrap them in descriptive functions in `domain/rules.py` (e.g. `peut_gerer_clinique(user, clinic)`).

* **Rule 20 (Safe Deletability):** If dropping a module's folder breaks an unrelated module, your domain boundaries are bleeding. Each module must be independently removable.

* **Rule 21 (No KEYS \* in Production Redis):** Use `SCAN` + `DEL` in batches. `KEYS *` blocks the Redis event loop and is forbidden in any environment with real data.

* **Rule 22 (Search Module Is Read-Only):** The `recherche` module contains no services and no write repository methods. It reads from its RediSearch index only. No writes, no mutations.

* **Rule 23 (Slot Scope Is Explicit):** Valid slot ownership combinations are: clinic-level (`service_id=null, subservice_id=null`), service-level (`service_id!=null, subservice_id=null`), or subservice-level (`service_id!=null, subservice_id!=null`). Missing identifiers are never inferred automatically.

---

# Anti-Patterns Reference

| If you see this | Module context | The fix |
| --- | --- | --- |
| `MedecinORM.objects.get()` inside a service | any | Move to `DjangoMedecinRepository`; import the repo class in the service |
| Business logic in `router.py` | any | Extract to a service class; router calls `service.executer()` |
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
| `from apps.medecins.models import MedecinORM` inside `rendez_vous/` | rendez_vous | Import `DjangoMedecinRepository` and use its read methods |
| `from apps.autorisations.models import RoleORM` in any service | any | Import `DjangoAutorisationRepository` instead |
| Global role assigned with no `clinic_id` scope | autorisations | All roles are per-clinic; `clinic_id` is always NOT NULL |
| `cache.keys("search:*")` or `KEYS search:*` in production | recherche | Use `SCAN` + `DEL` in batches |
| Cache invalidation inside `transaction.atomic()` | any | Move after the `with` block closes or register via `on_commit` |
| `duration_min` read from `ServiceORM` | any | `duration_min` lives on `SousServiceORM` exclusively |
| `vecteur_recherche` manually assigned in application code | sous_services / cliniques | The column is owned by a database-level trigger (PostgreSQL) — Python assignment is silently overwritten |
| `recherche` module importing a write repository or service | recherche | The `recherche` module is read-only by Rule 22 |
| Slot created with mixed or inferred scope | scheduling | Enforce the explicit scope matrix in the service layer |

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
