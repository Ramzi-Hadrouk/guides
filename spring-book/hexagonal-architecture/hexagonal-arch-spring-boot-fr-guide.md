# Hexagonal Architecture — Spring Boot Guide (French Client)

Mandatory architecture standard for AI coding agents generating Spring Boot code. Domain-agnostic, reusable across projects. Business content is in French (packages, classes, DB, API paths); documentation stays in English. Every rule is binding unless a human reviewer explicitly overrides it.

## Table of Contents

1. [Architecture Principles](#1-architecture-principles)
2. [Global Project Structure](#2-global-project-structure)
3. [Module Organization](#3-module-organization)
4. [Internal Structure of One Module](#4-internal-structure-of-one-module)
5. [Hexagonal Architecture Rules](#5-hexagonal-architecture-rules)
6. [Inter-Module Communication](#6-inter-module-communication)
7. [Naming Conventions](#7-naming-conventions)
8. [French Naming Rules](#8-french-naming-rules)
9. [Shared Package](#9-shared-package)
10. [Configuration Management](#10-configuration-management)
11. [Validation Rules](#11-validation-rules)
12. [Exception Handling](#12-exception-handling)
13. [Mapping Rules](#13-mapping-rules)
14. [Persistence Rules](#14-persistence-rules)
15. [REST API Standards](#15-rest-api-standards)
16. [Swagger and OpenAPI](#16-swagger-and-openapi)
17. [Logging](#17-logging)
18. [Security](#18-security)
19. [Testing](#19-testing)
20. [Folder Tree Example](#20-folder-tree-example)
21. [Complete Request Flow](#21-complete-request-flow)
22. [Required Components Checklist](#22-required-components-checklist)
23. [AI Agent Rules](#23-ai-agent-rules)
24. [Quality Checklist](#24-quality-checklist)

---

## 1. Architecture Principles

| Principle | Rule |
|---|---|
| Domain independence | `domain/` has zero import from Spring, JPA, Jackson, Lombok annotations that leak framework behavior, or any infrastructure library. Pure Java only. |
| Dependency inversion | Domain and application define interfaces (ports). Infrastructure implements them. Source code dependencies point inward, never outward. |
| SOLID | Every class respects Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion. Reject designs that violate any of them. |
| Clean Code | Small classes, small methods (≤ 30 lines as a guide), intention-revealing names, no dead code, no commented-out code. |
| Single Responsibility | One class, one reason to change. A class that both validates, maps, and persists must be split. |
| Constructor injection only | `@Autowired` on fields or setters is forbidden. All dependencies are `private final` fields set via constructor. Lombok `@RequiredArgsConstructor` is allowed. |
| Immutable DTOs | Requests, Responses, and Value Objects are Java `record` types or classes with `final` fields, no setters. |
| Composition over inheritance | Reuse behavior via delegation and injected collaborators. Inheritance is reserved for genuine polymorphic hierarchies (e.g. a domain exception hierarchy). |
| No circular dependencies | No package, class, or module may depend on itself transitively. Cross-module calls only through public application ports or domain events (see [Section 6](#6-inter-module-communication)). |
| Feature-first organization | The codebase is split by business capability (module) first, technical layer second. There is no project-wide `controllers/`, `services/`, or `repositories/` package. |

---
---

## 2. Global Project Structure

```
src/main/java/com/entreprise/projet
├── ProjetApplication.java
├── modules/
│   ├── utilisateurs/
│   ├── authentification/
│   ├── produits/
│   └── commandes/
├── shared/
└── configuration/
```

| Folder | Responsibility |
|---|---|
| `ProjetApplication.java` | Spring Boot entry point (`@SpringBootApplication`). Nothing else lives at this level. |
| `modules/` | Container for every business capability. Each subfolder is a complete, independent hexagon (see [Section 3](#3-module-organization) and [Section 4](#4-internal-structure-of-one-module)). |
| `shared/` | Project-wide, business-agnostic building blocks reused by every module (see [Section 9](#9-shared-package)). |
| `configuration/` | Cross-cutting Spring configuration that is not owned by a single module: security filter chain, CORS, OpenAPI bean, async/thread pool config, Jackson global config, web MVC config. |

**Do not** replicate the naive `application / domain / infrastructure` split at the project root. A single global `domain/` package for every entity in the system defeats feature isolation, creates a god-package, and breaks module independence. `application`, `domain`, and `infrastructure` only exist **inside** each module (Section 4). This is a deliberate deviation from a purely layered layout in favor of modular hexagonal design, and it is mandatory.

---
---

## 3. Module Organization

Every module is a self-contained hexagon: it owns its API, its use cases, its domain model, and its persistence adapters. A module never reaches into another module's internal packages.

| Module (French) | Business meaning |
|---|---|
| `utilisateurs` | Users |
| `authentification` | Authentication / identity |
| `produits` | Products / catalog |
| `commandes` | Orders |

Rules:

- A module MUST contain, at minimum: `api`, `application`, `domain`, `infrastructure` (Section 4).
- A module MUST NOT import another module's `domain`, `application`, or `infrastructure` classes directly.
- A module MAY depend on `shared/` and `configuration/`.
- A new business capability = a new module under `modules/`, never a new package bolted onto an existing module.
- Module names are French, singular-concept-plural-folder (e.g. `utilisateurs`, not `userManagement`).

---
---

## 4. Internal Structure of One Module

```
modules/utilisateurs/
├── api/
│   ├── controllers/
│   ├── requests/
│   ├── responses/
│   └── documentation/
├── application/
│   ├── ports/
│   │   ├── in/
│   │   └── out/
│   ├── services/
│   ├── mappers/
│   └── validators/
├── domain/
│   ├── entities/
│   ├── valueobjects/
│   ├── aggregates/
│   ├── repositories/
│   ├── events/
│   ├── exceptions/
│   └── domainservices/
└── infrastructure/
    ├── persistence/
    │   ├── entities/
    │   ├── repositories/
    │   └── adapters/
    └── configuration/
```

> `shared/` is **not** duplicated inside each module. A single project-level `shared/` (Section 9) is used by every module. If a module tree in a spec or ticket shows a local `shared/`, treat it as a mistake and use the project-level one instead.

| Folder | Responsibility |
|---|---|
| `api/controllers` | `@RestController` classes. HTTP binding only — no business logic. |
| `api/requests` | Inbound DTOs (records) with Bean Validation annotations. |
| `api/responses` | Outbound DTOs (records). Never a domain or JPA entity. |
| `api/documentation` | OpenAPI grouping: `package-info.java` for the module's public API, reusable `@ExampleObject` payloads. |
| `application/ports/in` | Input ports: one interface per use case, called by controllers. |
| `application/ports/out` | Output ports for external needs that are **not** persistence (notification, payment gateway, external API client, message broker). |
| `application/services` | Use case implementations — one class per `ports/in` interface. Orchestrate domain objects and ports. Hold `@Transactional` boundaries. |
| `application/mappers` | MapStruct interfaces: Request → domain command, domain result → Response. |
| `application/validators` | Cross-field / business-rule validators invoked before a use case executes. |
| `domain/entities` | Mutable domain objects with identity. No framework annotations. |
| `domain/valueobjects` | Immutable objects without identity (e.g. `Adresse`, `Email`). |
| `domain/aggregates` | Aggregate roots enforcing invariants across a cluster of entities/value objects. |
| `domain/repositories` | Repository interfaces (persistence output ports), owned by the domain since they express what the domain needs persisted. No Spring/JPA imports. |
| `domain/events` | Domain events raised by aggregates (e.g. `UtilisateurCreeEvent`). |
| `domain/exceptions` | Business rule violations, thrown by entities/aggregates/domain services. |
| `domain/domainservices` | Stateless domain logic that doesn't naturally belong to one entity. |
| `infrastructure/persistence/entities` | JPA `@Entity` classes. Separate from `domain/entities`. |
| `infrastructure/persistence/repositories` | Spring Data JPA interfaces (`JpaRepository<...>`), used only by the adapter in the same package — never injected into `application` or `domain`. |
| `infrastructure/persistence/adapters` | Implementations of `domain/repositories` interfaces. Convert JPA entity ↔ domain entity via a mapper. |
| `infrastructure/configuration` | Module-scoped `@Configuration` (e.g. a module-specific `@EnableJpaRepositories` base package, module beans). |

> No separate `usecases/` folder. `ports/in` is the contract, `application/services` is the one implementation of each contract (class name ends in `Service` — Section 7). A third folder would only duplicate one of the two; don't create it.

---
---

## 5. Hexagonal Architecture Rules

| Component | Responsibility |
|---|---|
| Inbound adapter (Controller) | Translate HTTP ↔ application input. No business rules. |
| Outbound adapter (JPA adapter, external client) | Implement an output port. Translate application/domain calls into technology-specific calls. |
| Input port | Interface describing a use case as the application exposes it (`CreerUtilisateurUseCase.executer(...)`). |
| Output port | Interface describing what the application/domain needs from the outside world (`UtilisateurRepository`, `NotificationPort`). |
| Use case | One class = one business operation. Implements exactly one input port. |
| Application service | Synonym for use case implementation. Coordinates domain objects, ports, transactions. Contains **no** business rules of its own — those live in the domain. There is no separate "generic service" category: logic needed by more than one use case is either a `domainservice` (if it's a business rule) or a small collaborator injected into both use case implementations (if it's plumbing). |
| Domain service | Business logic that spans multiple entities/aggregates and doesn't belong to a single one. |
| Entity / Aggregate | Enforces its own invariants. Throws domain exceptions on violation. |
| Repository (interface) | Declares persistence operations the domain needs, in domain language (`rechercherParEmail`, not `findByEmailNative`). |
| Mapper | Pure conversion, no logic, no validation, no side effects. |
| Validator | Rejects invalid input before it reaches domain logic. |
| Configuration | Wires beans, no business logic. |
| Infrastructure | Implements ports. Knows about the domain; the domain never knows about it. |

**Allowed dependency direction:**

```
api  →  application  →  domain
infrastructure  →  application (implements ports/out)
infrastructure  →  domain (implements domain/repositories)
```

`domain` depends on nothing inside the module. `application` depends only on `domain`. `infrastructure` depends on `application` and `domain` interfaces, never the reverse.

**Forbidden dependencies:**

| Forbidden | Why |
|---|---|
| `domain` → `infrastructure` | Breaks dependency inversion. |
| `domain` → `application` | Domain must not know use cases exist. |
| `domain` → Spring / JPA / Jackson | Domain must stay framework-free. |
| `api.controllers` → `infrastructure` | Controllers only know ports/use cases. |
| `api.controllers` → `domain.repositories` | Never inject a repository into a controller. |
| module A → module B `domain`/`application`/`infrastructure` | Modules communicate only through public contracts (Section 6). |

---
---

## 6. Inter-Module Communication

- A module never imports another module's `domain`, `application`, or `infrastructure` package.
- Cross-module reads/writes go through the target module's public input port (its use case interfaces), treated as that module's public API.
- Cross-module reactions to something that happened elsewhere use domain events (`domain/events`) published by the source module and consumed by a listener in the target module's `infrastructure` or `application` layer — not a direct method call chain.
- Shared data types crossing a module boundary are plain, framework-free DTOs (not the other module's domain entity) — define a small contract type if needed, or reuse a type from `shared/` if it is truly generic.
- Circular module dependencies (`utilisateurs` → `commandes` → `utilisateurs`) are forbidden. If two modules need each other, extract the shared concept into its own module or into `shared/`.
---

## 7. Naming Conventions

| Element | Pattern | Example |
|---|---|---|
| Use case (input port) | `<Verbe><Entite>UseCase` | `CreerUtilisateurUseCase` |
| Use case implementation | `<Verbe><Entite>Service` | `CreerUtilisateurService` |
| Output port | `<Entite>Port` or `<Entite>Repository` for persistence | `NotificationPort`, `UtilisateurRepository` |
| Outbound adapter | `<Entite><Technologie>Adapter` | `UtilisateurJpaAdapter` |
| Controller | `<Entite>Controller` | `UtilisateurController` |
| Repository interface | `<Entite>Repository` | `UtilisateurRepository` |
| Spring Data JPA repository | `<Entite>JpaRepository` | `UtilisateurJpaRepository` |
| Domain entity | `<Entite>` | `Utilisateur` |
| JPA entity | `<Entite>JpaEntity` | `UtilisateurJpaEntity` |
| DTO — Request | `<Verbe><Entite>Requete` | `CreerUtilisateurRequete` |
| DTO — Response | `<Entite>Reponse` | `UtilisateurReponse` |
| Exception | `<Raison>Exception` | `UtilisateurNonTrouveException` |
| Domain event | `<Entite><ParticipePasse>Event` | `UtilisateurCreeEvent` |
| Validator | `<Cible>Validator` | `CreerUtilisateurValidator` |
| Configuration class | `<Domaine>Configuration` | `SecuriteConfiguration` |
| Mapper | `<Entite>Mapper` | `UtilisateurMapper` |
| Specification | `<Entite>Specification` | `UtilisateurSpecification` |
| Factory | `<Entite>Factory` | `CommandeFactory` |
| Assembler | `<Entite>Assembler` | `CommandeAssembler` |

Rule: class names are French for the business term, English for the technical suffix (`UseCase`, `Service`, `Repository`, `Adapter`, `Controller`, `Mapper`, `Exception`, `Event`, `Configuration`, `Validator`, `Specification`, `Factory`, `Assembler` stay in English — only the business noun/verb is French). This keeps the pattern recognizable while satisfying the French-content rule in [Section 8](#8-french-naming-rules).

---
---

## 8. French Naming Rules

This project targets a French client. French is mandatory for business-facing identifiers; English is mandatory for documentation artifacts.

**French (mandatory):**

| Concept | Convention | Example |
|---|---|---|
| Package (module) | lowercase French noun, plural | `modules.utilisateurs`, `modules.commandes` |
| Class (entity) | French noun | `Utilisateur`, `Commande`, `LigneCommande` |
| Interface (port) | French verb/noun + English suffix | `CreerUtilisateurUseCase` |
| Variable | French, camelCase | `nomUtilisateur`, `dateCreation` |
| Method | French verb, camelCase | `creer()`, `rechercherParEmail()`, `annuler()` |
| DTO — Request | French verb + entity + `Requete` | `CreerUtilisateurRequete` |
| DTO — Response | French entity + `Reponse` | `UtilisateurReponse` |
| Service | French verb + entity + `Service` (Section 7) | `CreerUtilisateurService` |
| Controller | French entity + `Controller` | `UtilisateurController` |
| Port | French entity/action + `Port`/`UseCase` | `NotificationPort` |
| Adapter | French entity + technology + `Adapter` | `UtilisateurJpaAdapter` |
| Entity (domain/JPA) | French noun | `Utilisateur`, `UtilisateurJpaEntity` |
| Enum | French noun, French values | `StatutCommande { EN_ATTENTE, VALIDEE, ANNULEE }` |
| Constant | French, UPPER_SNAKE_CASE | `TAILLE_PAGE_PAR_DEFAUT` |
| Exception | French reason + `Exception` | `UtilisateurNonTrouveException` |
| Event | French entity + participe passé + `Event` | `UtilisateurCreeEvent` |
| Database table | French, snake_case, plural | `utilisateurs`, `lignes_commande` |
| Database column | French, snake_case | `nom_utilisateur`, `date_creation` |
| API path | French, kebab-case, plural | `/api/v1/utilisateurs`, `/api/v1/lignes-commande` |

**English (mandatory, no exceptions):**

- Code comments and Javadoc.
- This document and any other Markdown documentation.
- README files.
- Commit messages.
- Architecture decision records / design docs.

Technical suffixes (`Service`, `Controller`, `Repository`, `Adapter`, `UseCase`, `Port`, `Mapper`, `Exception`, `Event`, `Configuration`, `Validator`, `Specification`, `Factory`, `Assembler`, `Requete`, `Reponse`) stay in English as defined in Section 7 — they are structural markers, not business vocabulary, and remain consistent across the whole codebase.

---
---

## 9. Shared Package

```
shared/
├── exceptions/
├── constants/
├── utils/
└── enums/
```

| Belongs in `shared/` | Never in `shared/` |
|---|---|
| `GlobalExceptionHandler` (`@RestControllerAdvice`) | Any module-specific entity, DTO, or use case |
| Base `ProblemDetail` / `ErrorResponse` wrapper | Business rules or validation specific to one module |
| `AppConstants` (non-business technical constants) | A module's repository or controller |
| `DateUtils`, `StringUtils`, technical helpers | Anything that changes when one module's business rules change |
| Generic enums reused everywhere (`Statut`, `Devise`) | A module-specific enum (`StatutCommande` belongs in `modules/commandes/domain`) |
| `PageRequete` / `PageResultat` (framework-free pagination wrapper) | `org.springframework.data.domain.Pageable`/`Page` leaking past infrastructure |
| `TriRequete` (framework-free sort wrapper) | — |
| `AuditableEntity` base class (createdAt/updatedAt fields) | — |
| `CurrentUserProvider` interface (security helper port) | — |
| Global MapStruct configuration (`@MapperConfig`) | — |

Rule: if removing a class from `shared/` would only break one module, it does not belong in `shared/` — move it into that module.

---
---

## 10. Configuration Management

- Never call `System.getenv()` or inject `@Value` scattered across arbitrary classes.
- Centralize every environment-driven setting in a typed `@ConfigurationProperties` class, validated with `@Validated`.
- One configuration class per concern, placed in `configuration/` (global) or `modules/<module>/infrastructure/configuration/` (module-scoped).

```java
@ConfigurationProperties(prefix = "app.securite.jwt")
@Validated
public record JwtProperties(
    @NotBlank String secret,
    @Positive long dureeAccesMinutes,
    @Positive long dureeRafraichissementJours
) {}
```

- `application.yml` — defaults common to all environments.
- `application-dev.yml` — local/dev overrides, verbose logging, permissive CORS.
- `application-prod.yml` — production values, secrets referenced via environment variables or a vault, never hardcoded.
- Activate profiles via `SPRING_PROFILES_ACTIVE`, never hardcode `spring.profiles.active` in a committed file for prod.
- Secrets (DB password, JWT secret, API keys) come from environment variables or a secret manager, injected into `.yml` via `${VAR_NAME}`. Never commit a real secret.
- Fail fast: an invalid or missing required property must fail application startup, not fail silently at runtime.

---
---

## 11. Validation Rules

| Layer | Validation type | Tool |
|---|---|---|
| `api/requests` | Structural/input validation (format, required fields, size) | Bean Validation (`@NotNull`, `@Email`, `@Size`, ...) on the record, `@Valid` on the controller parameter |
| `application/validators` | Cross-field or use-case-specific business validation that doesn't require loading full domain state | Custom `Validator`/`ConstraintValidator`, or a plain validator class called by the use case |
| `domain` | Invariants — rules that must always hold for the object to exist | Enforced in entity/value object constructors and factory methods; throw a domain exception, never return `false`/`null` |
| Cross-field | e.g. `dateFin` must be after `dateDebut` | Custom class-level `@ConstraintValidator` |
| Create vs update | Different required fields per operation | Bean Validation groups (`OnCreate`, `OnUpdate`) |

Rule: validation that can be expressed with an annotation stays an annotation. Validation that requires business knowledge belongs in `application/validators` or the domain — never duplicated in the controller.

---
---

## 12. Exception Handling

```
DomainException (abstract, unchecked)
├── EntiteNonTrouveeException
├── RegleMetierVioleeException
└── EtatInvalideException

ApplicationException (abstract, unchecked)
└── OperationNonAutoriseeException

InfrastructureException (abstract, unchecked)
├── PersistanceException
└── ServiceExterneException
```

- Every module-specific exception extends one of these three shared base classes from `shared/exceptions`.
- `domain/exceptions` holds the concrete subclasses raised by that module's domain (`UtilisateurNonTrouveException extends EntiteNonTrouveeException`).
- REST exposure goes through a single `GlobalExceptionHandler` (`@RestControllerAdvice`) in `shared/`, mapping exception types to HTTP status and a `ProblemDetail` (RFC 7807) body.

```json
{
  "type": "https://api.projet.fr/erreurs/utilisateur-non-trouve",
  "title": "Utilisateur non trouvé",
  "status": 404,
  "detail": "Aucun utilisateur avec l'identifiant 42",
  "instance": "/api/v1/utilisateurs/42"
}
```

- Controllers never catch exceptions to build error responses manually — that is the handler's job.
- Never leak a stack trace, SQL error, or internal class name in a response body.

---
---

## 13. Mapping Rules

- Never expose a domain entity or a JPA entity through the API. Every boundary crossing goes through a mapper.
- Request → domain: `application/mappers` converts an `api/requests` DTO into a domain object or use-case command.
- Domain → Response: `application/mappers` converts a domain result into an `api/responses` DTO.
- Domain ↔ JPA entity: `infrastructure/persistence/adapters` (via a mapper) converts between `domain/entities` and `infrastructure/persistence/entities`.
- Mappers do pure conversion only — no validation, no business logic, no side effects, no calls to other services.
- Use **MapStruct** for every mapper (`@Mapper(componentModel = "spring")`). Hand-written mappers are only acceptable for trivial one-field cases.

```java
@Mapper(componentModel = "spring")
public interface UtilisateurMapper {
    Utilisateur versDomaine(CreerUtilisateurRequete requete);
    UtilisateurReponse versReponse(Utilisateur utilisateur);
}
```

---
---

## 14. Persistence Rules

Repository interface/adapter placement is defined in [Section 4](#4-internal-structure-of-one-module) and [Section 5](#5-hexagonal-architecture-rules) — not repeated here. This section covers persistence-specific behavior only:

- `infrastructure/persistence/entities` holds the `@Entity`-annotated classes. They are never returned outside `infrastructure`.
- Transaction boundaries: on the use case implementation (`application/services`, `@Transactional`) only — never on the controller, never on the repository adapter.
- Use `@Version` on any JPA entity that can be concurrently updated (optimistic locking).
- Dynamic/conditional queries use JPA `Specification<T>` inside `infrastructure/persistence/adapters` (or a `specifications/` subfolder if a module has several), never raw string-concatenated JPQL.
- Pagination/sorting: `Pageable`, `Page<T>`, `Sort` stop at `infrastructure`. The adapter converts them to/from the framework-free `PageRequete`/`PageResultat`/`TriRequete` wrappers defined in [Section 9](#9-shared-package) before returning to `application`/`domain`.

---
---

## 15. REST API Standards

- Controllers: bind the request, call exactly one input port, map the result, return. No `if`/business branching beyond HTTP status selection.

| Situation | HTTP status |
|---|---|
| Successful GET | 200 |
| Successful POST (creation) | 201 (with `Location` header) |
| Successful action with no body to return | 204 |
| Validation failure | 400 |
| Not authenticated | 401 |
| Authenticated but not authorized | 403 |
| Resource not found | 404 |
| Conflict (duplicate, concurrent update) | 409 |
| Semantically invalid request | 422 |
| Unhandled/infrastructure failure | 500 |

- Versioning: URI-based, `/api/v1/...`. Breaking changes ship as `/api/v2/...`, old version kept until deprecation window closes.
- Pagination: query params `page`, `taille`, response includes `pageCourante`, `tailleTotale`, `nombreElements`.
- Filtering/sorting: query params (`tri=nom,asc`), never in the path.
- Errors: always `ProblemDetail` (Section 12).
- Naming: resource paths are French, plural, kebab-case: `/api/v1/utilisateurs`, `/api/v1/commandes/{id}/lignes-commande`.

---
---

## 16. Swagger and OpenAPI

Mandatory on every public endpoint (SpringDoc OpenAPI):

| Annotation | Required for |
|---|---|
| `@Tag` | Controller class — module grouping |
| `@Operation` | Every endpoint — summary + description |
| `@ApiResponse` (one per possible status) | Every endpoint |
| `@Parameter` | Every path/query parameter |
| `@Schema` | Every request/response field with unclear semantics |
| `@ExampleObject` | At least one realistic example per endpoint |
| `@SecurityRequirement` | Every protected endpoint |

Checklist per endpoint: operation documented, all parameters documented, all response codes documented with examples, error response documented, security scheme declared, request/response schemas fully typed (no raw `Object`/`Map`).

---
---

## 17. Logging

- SLF4J only (`org.slf4j.Logger`). `System.out.println` / `printStackTrace` are forbidden — treat as a build failure.
- Structured (JSON) logging in prod via a Logback JSON encoder.
- Every inbound request gets/propagates a correlation ID via MDC (`X-Correlation-Id` header in, same header out), included automatically in every log line for that request.
- Never log passwords, tokens, JWTs, full card numbers, or other secrets/PII — mask or omit.

| Level | Use for |
|---|---|
| ERROR | Unexpected failure requiring attention (infra failure, unhandled exception) |
| WARN | Recoverable/unexpected but handled situation (retry, fallback used) |
| INFO | Significant business/application events (use case executed, module started) |
| DEBUG | Detailed flow useful in dev/troubleshooting, disabled in prod by default |
| TRACE | Very fine-grained diagnostic detail, disabled by default everywhere |

---
---

## 18. Security

- Authentication: JWT (access + refresh token). Access token short-lived, refresh token rotated.
- Password hashing: BCrypt, strength ≥ 10, never a reversible or fast hash (no MD5/SHA1/plain SHA256 for passwords).
- Authorization: method-level (`@PreAuthorize`) on use case implementations, not scattered `if` checks in controllers.
- Input sanitization: every free-text field validated/sanitized before persistence or use in a query; parameterized queries only, never string-concatenated SQL/JPQL.
- CORS: explicit allow-list of origins per environment, no `*` in production.
- CSRF: disabled for stateless JWT bearer-token APIs (documented as an explicit decision); enabled if any cookie-based session auth is used.
- Rate limiting: recommended at the gateway/reverse-proxy layer, or via Bucket4j in-app for sensitive endpoints (login, password reset).
- Never trust client-supplied roles/IDs — always resolve identity from the authenticated principal.

---
---

## 19. Testing

| Test type | Scope | Tooling | Naming |
|---|---|---|---|
| Unit | Use case, domain entity, validator, mapper — in isolation | JUnit 5 + Mockito, mock ports only | `<Classe>Test` |
| Integration | Full Spring context, real behavior across layers | `@SpringBootTest` + Testcontainers | `<Classe>IT` |
| Repository | JPA adapter against a real database | `@DataJpaTest` + Testcontainers (Postgres) | `<Adapter>IT` |
| Controller | HTTP layer | `@WebMvcTest` + `MockMvc` (or full IT) | `<Controller>Test` |

- Mock only interfaces (ports). Never mock a domain entity, value object, or a concrete mapper.
- Testcontainers is mandatory for any test touching a real database — no H2-as-Postgres-substitute for anything beyond trivial smoke tests.
- Coverage targets (Jacoco, enforced in CI): domain ≥ 90%, application ≥ 85%, overall ≥ 80%.
- Every use case has at least: one happy-path test, one domain-rule-violation test, one not-found/edge-case test.

### Test Folder Organization

`src/test/java` mirrors `src/main/java` exactly — same base package, same module, same layer subfolders. A production class and its test(s) always live at the same relative path.

```
src/test/java/com/entreprise/projet
├── modules/
│   ├── utilisateurs/
│   │   ├── api/controllers/
│   │   │   └── UtilisateurControllerTest.java
│   │   ├── application/
│   │   │   ├── services/
│   │   │   │   └── CreerUtilisateurServiceTest.java
│   │   │   └── validators/
│   │   │       └── CreerUtilisateurValidatorTest.java
│   │   ├── domain/
│   │   │   └── entities/
│   │   │       └── UtilisateurTest.java
│   │   └── infrastructure/persistence/adapters/
│   │       └── UtilisateurRepositoryJpaAdapterIT.java
│   └── commandes/
│       └── ... (same shape)
└── shared/
    └── testsupport/
        ├── containers/     → PostgresContainerBaseIT (one shared Testcontainers base, extended by every *IT)
        ├── fixtures/       → one test-data builder per aggregate root (e.g. UtilisateurTestBuilder)
        └── stubs/          → hand-written fakes for output ports, for unit tests where a Mockito mock is overkill
```

Rules:

- One test class per production class, named per the table above (`Test` for unit/controller-slice, `IT` for anything hitting Spring context or a database).
- No module reaches into another module's test folder. Shared test helpers go in `shared/testsupport`, mirroring the `shared/` rule in [Section 9](#9-shared-package).
- Testcontainers setup is defined **once** (`shared/testsupport/containers`) and reused via inheritance or a shared static container — never re-declared per module.
- Test-data builders live next to the aggregate they build, one per aggregate root, reused across unit and integration tests instead of duplicating object construction in every test method.
- A module's test folder must contain at least one test per file listed in its [Required Components Checklist](#22-required-components-checklist) entry.

---
---

## 20. Folder Tree Example

Realistic, fully-populated tree for the `commandes` (orders) module:

```
modules/commandes/
├── api/
│   ├── controllers/
│   │   └── CommandeController.java
│   ├── requests/
│   │   ├── CreerCommandeRequete.java
│   │   └── AjouterLigneCommandeRequete.java
│   ├── responses/
│   │   ├── CommandeReponse.java
│   │   └── LigneCommandeReponse.java
│   └── documentation/
│       └── package-info.java
├── application/
│   ├── ports/
│   │   ├── in/
│   │   │   ├── CreerCommandeUseCase.java
│   │   │   └── AnnulerCommandeUseCase.java
│   │   └── out/
│   │       └── NotificationPort.java
│   ├── services/
│   │   ├── CreerCommandeService.java
│   │   └── AnnulerCommandeService.java
│   ├── mappers/
│   │   └── CommandeMapper.java
│   └── validators/
│       └── CreerCommandeValidator.java
├── domain/
│   ├── entities/
│   │   └── LigneCommande.java
│   ├── valueobjects/
│   │   └── Montant.java
│   ├── aggregates/
│   │   └── Commande.java
│   ├── repositories/
│   │   └── CommandeRepository.java
│   ├── events/
│   │   └── CommandeCreeeEvent.java
│   ├── exceptions/
│   │   └── CommandeNonTrouveeException.java
│   └── domainservices/
│       └── CalculateurTotalCommande.java
└── infrastructure/
    ├── persistence/
    │   ├── entities/
    │   │   ├── CommandeJpaEntity.java
    │   │   └── LigneCommandeJpaEntity.java
    │   ├── repositories/
    │   │   └── CommandeJpaRepository.java
    │   └── adapters/
    │       └── CommandeRepositoryJpaAdapter.java
    └── configuration/
        └── CommandesConfiguration.java
```

---
---

## 21. Complete Request Flow

```
HTTP POST /api/v1/commandes
        │
        ▼
CommandeController.creerCommande(CreerCommandeRequete)
        │  (@Valid triggers Bean Validation)
        ▼
CommandeMapper.versDomaine(requete)  →  command object
        │
        ▼
CreerCommandeUseCase.executer(command)      [input port]
        │
        ▼
CreerCommandeService.executer(command)      [use case impl, @Transactional]
        │  - builds/validates the Commande aggregate (domain rules enforced here)
        │  - raises CommandeCreeeEvent
        ▼
CommandeRepository.enregistrer(commande)    [output port, domain]
        │
        ▼
CommandeRepositoryJpaAdapter.enregistrer(commande)   [infrastructure]
        │  - maps domain Commande → CommandeJpaEntity
        ▼
CommandeJpaRepository.save(entity)          [Spring Data JPA]
        │
        ▼
Database (commandes, lignes_commande tables)
        │
        ▼
CommandeRepositoryJpaAdapter maps entity → domain Commande, returns it
        │
        ▼
CreerCommandeService returns domain result to controller
        │
        ▼
CommandeMapper.versReponse(commande)  →  CommandeReponse
        │
        ▼
HTTP 201 Created, Location: /api/v1/commandes/{id}, body: CommandeReponse
```

---
---

## 22. Required Components Checklist

Every module must contain, for each use case it exposes:

- ☐ Controller
- ☐ Request DTO
- ☐ Response DTO
- ☐ Mapper
- ☐ Use case (input port + implementation)
- ☐ Output port(s) it depends on
- ☐ Adapter(s) implementing those output ports
- ☐ Repository interface (if persistence involved)
- ☐ Validation (Bean Validation + business validator where needed)
- ☐ Domain/application/infrastructure exceptions as applicable
- ☐ Unit tests + integration test
- ☐ Swagger annotations
- ☐ Logging at key decision/error points
- ☐ Configuration externalized (no hardcoded values)
- ☐ `package-info.java` / short module documentation

---
---

## 23. AI Agent Rules

**NEVER:**

- Skip validation, Swagger, logs, mappers, or exceptions "to save time."
- Put business logic in a controller.
- Inject a repository directly into a controller.
- Put a Spring, JPA, or Jackson annotation inside `domain/`.
- Return or accept a JPA entity at the API boundary.
- Duplicate logic that already exists in `shared/` or another use case.
- Ignore the module/layer folder structure defined in this guide.
- Hardcode a configuration value that should be externalized.
- Call `System.getenv()` or read a raw environment variable outside a typed configuration class.
- Ignore null-safety (return `null` instead of `Optional`, throw, or a domain exception).
- Create a "God service" handling more than one use case's orchestration.
- Cram every method into one `Service` class instead of one use case per class.
- Let one module import another module's `domain`/`application`/`infrastructure` package.

**ALWAYS:**

- Respect the hexagonal boundaries and dependency direction in Section 5.
- Keep business rules inside the domain (entities/aggregates/domain services), orchestration inside use cases.
- Use constructor injection exclusively.
- Create dedicated Request/Response DTOs — never reuse a domain or JPA entity as a DTO.
- Write validation at the correct layer (Section 11).
- Create a mapper for every boundary crossing (Section 13).
- Write unit and integration tests for every use case.
- Handle exceptions through the shared hierarchy and global handler (Section 12).
- Document every endpoint with OpenAPI annotations.
- Externalize configuration through typed `@ConfigurationProperties` classes.
- Follow the naming conventions in Sections 7 and 8 exactly.
- Keep modules independent; communicate cross-module only per Section 6.

---
---

## 24. Quality Checklist

Run this before marking a feature complete. Each item is a concrete check, not a restatement of a rule — if an item fails, fix it and re-run the whole list once.

**Architecture**
- ☐ `domain` compiles with zero Spring/JPA/Jackson imports.
- ☐ No import from `api`/`infrastructure` into `domain` or `application` (grep for it).
- ☐ No import of another module's `domain`/`application`/`infrastructure` package (Section 6).
- ☐ No two modules import each other (circular module dependency check).

**Naming**
- ☐ Every new class name matches its Section 7 pattern exactly (right suffix, right casing).
- ☐ Every business identifier (class, field, DB column, API path) is French; every comment/Javadoc is English (Section 8).

**Validation**
- ☐ Every field on every request DTO has an explicit Bean Validation annotation, or a documented reason it doesn't.
- ☐ Business-rule validation lives in `application/validators` or the domain — not re-checked again in the controller.

**Exceptions**
- ☐ Every `throw` in the module extends `DomainException`, `ApplicationException`, or `InfrastructureException`.
- ☐ `GlobalExceptionHandler` has a mapping for each new exception type, verified with a test that asserts the HTTP status.

**Logging**
- ☐ Zero `System.out`/`printStackTrace` in a `grep` of the module.
- ☐ Every catch of an unexpected exception logs at `ERROR` with the correlation ID present in the log line.
- ☐ No password, token, or full PII field appears in any log statement.

**Swagger**
- ☐ Every new/changed endpoint has `@Operation`, one `@ApiResponse` per status it can return, and at least one `@ExampleObject`.
- ☐ Rendered OpenAPI spec (`/v3/api-docs`) contains no `Object`/`Map` schema for a field that has a real shape.

**Tests**
- ☐ Test files exist at the mirrored path under `src/test/java` for every new production class (Section 19).
- ☐ One happy-path + one domain-rule-violation + one not-found/edge-case test per use case.
- ☐ Module coverage meets the Section 19 thresholds in the Jacoco report, not just "tests pass."

**Configuration**
- ☐ `grep` for `@Value` and `System.getenv` in the module returns nothing.
- ☐ Every new setting has a field in a `@Validated @ConfigurationProperties` class, present in `application.yml` with a safe dev default.

**Security**
- ☐ Every new endpoint has an explicit `@PreAuthorize` (or is deliberately public, documented as such).
- ☐ Any endpoint handling login/reset/high-cost operations has rate limiting applied (Section 18).
- ☐ No string-concatenated query anywhere in the module.

**Documentation**
- ☐ `package-info.java` describes the module's public contract and is current with the latest change.

**Mappers**
- ☐ No method in `api` or `application` returns a type from `infrastructure/persistence/entities` or a raw JPA entity.
- ☐ Every mapper method is pure — no repository/service call inside a mapper.

**Ports**
- ☐ Each `ports/in` interface has exactly one implementation in `services`.
- ☐ Each `ports/out` interface is implemented by exactly one adapter; no unused port left behind.

**Adapters**
- ☐ Every adapter implements a declared port — no adapter called directly by `application`/`domain` by concrete type.

**Transactions**
- ☐ `@Transactional` appears only in `application/services`; `grep` confirms it's absent from controllers and adapters.
- ☐ A use case that writes to more than one aggregate is covered by a test proving atomicity (rollback on partial failure).

**DTOs**
- ☐ Every request/response DTO is a `record` (or fully immutable) — no setter exists on any DTO.
- ☐ No DTO field type is a domain entity, JPA entity, `Pageable`, or `Page`.

**Repositories**
- ☐ Interface in `domain/repositories`, zero JPA/Spring imports in that interface.
- ☐ Adapter in `infrastructure/persistence/adapters` is the only class implementing it.

**Packaging**
- ☐ `modules/<module>` matches the Section 4 tree exactly — no root-level `usecases/`, no per-module `shared/`, no missing required folder.

**Code quality**
- ☐ No dead code, no commented-out code, no unresolved `TODO` without a ticket reference.
- ☐ Static analysis (e.g. Checkstyle/SpotBugs, whatever the project uses) runs clean on the module.

---
