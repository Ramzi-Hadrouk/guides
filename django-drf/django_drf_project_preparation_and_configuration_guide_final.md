# Django REST API — Guide 1: Project Preparation & Configuration

**Mandatory engineering standard for preparing and configuring a production-capable Django REST Framework project.**

This guide covers the foundation that should be established **before feature development begins**.

It is intentionally project-agnostic. It does not define business modules, domain boundaries, use-case architecture, or feature organization. Those decisions belong to the development guide that follows.

The goal is to make the Django/DRF runtime predictable, secure, testable, observable, and ready for development.

---

## Index

- Chapter 1 — Project preparation
  - 1.1 Technology baseline
  - 1.2 Python and dependency management
  - 1.3 Environment configuration
  - 1.4 Settings structure
  - 1.5 Core Django settings
  - 1.6 Database configuration
  - 1.7 Django REST Framework configuration
  - 1.8 Authentication configuration
  - 1.9 CORS and CSRF
  - 1.10 Security settings
  - 1.11 URLs and API base path
  - 1.12 Serialization and rendering defaults
  - 1.13 Pagination, filtering, and ordering defaults
  - 1.14 API error handling
  - 1.15 API documentation
  - 1.16 Static files and media
  - 1.17 Time zone and localization
  - 1.18 Email
  - 1.19 Cache and Redis
  - 1.20 Background jobs
  - 1.21 Logging and observability
  - 1.22 Health and readiness endpoints
  - 1.23 Development settings
  - 1.24 Production settings
  - 1.25 Test settings
  - 1.26 ASGI / WSGI
  - 1.27 Dependency and security checks
  - 1.28 Docker and deployment configuration
  - 1.29 CI baseline
  - 1.30 Initial project validation

- Chapter 2 — Strict Configuration Checklist

---

# Chapter 1 — Project Preparation

## 1.1 Technology baseline

The default backend stack is:

- Python;
- Django;
- Django REST Framework;
- PostgreSQL for production;
- Redis only when a concrete feature requires caching or a broker;
- Celery only when asynchronous/background execution is actually required.

Optional infrastructure may include:

- object storage;
- reverse proxy/load balancer;
- database connection pooling;
- error tracking;
- metrics;
- distributed tracing;
- containerization;
- CI/CD.

Do not add infrastructure merely because it is commonly used by "enterprise" applications.

### Database policy

PostgreSQL is the default production database.

SQLite may be used for:

- quick local experiments;
- isolated tests that do not depend on PostgreSQL behavior.

Tests involving the following must run against PostgreSQL:

- constraints;
- transactions;
- locking;
- PostgreSQL-specific indexes;
- JSONB behavior;
- generated columns;
- concurrent updates;
- production SQL behavior.

Do not promise database portability unless portability is an actual requirement.

---

## 1.2 Python and dependency management

Use a project-level dependency configuration such as `pyproject.toml`.

Recommended structure:

```text
backend/
├── config/
├── apps/
├── tests/
├── manage.py
├── pyproject.toml
├── .env.example
├── .gitignore
└── README.md
```

### Python version

Pin a supported Python minor version for the project.

The development, CI, and production environments should use the same Python minor version unless there is a documented reason not to.

### Dependency rules

Separate dependencies conceptually into:

```text
Runtime
Development
Testing
Deployment / operational tooling
```

Do not install libraries "just in case".

Every dependency should have a concrete purpose.

### Dependency locking

The project MUST use a reproducible dependency installation strategy.

The exact mechanism may be:

- a lock file supported by the chosen package manager;
- pinned requirements;
- another reproducible dependency workflow.

A fresh environment should install the same dependency versions used by CI and deployment.

---

## 1.3 Environment configuration

Configuration must be externalized from source code.

Recommended flow:

```text
Environment variables / secret manager
                ↓
        configuration loader
                ↓
       validated Django settings
                ↓
            application
```

### Rules

- Real secrets are never committed.
- `.env.example` documents expected variables without real secrets.
- Business code must not read environment variables directly.
- Configuration should be read centrally from the settings layer.
- Required production configuration must fail fast when missing or invalid.
- Environment-specific values must not be hard-coded.
- Secret values must never appear in logs.
- Production configuration should come from environment/secret management rather than from source-controlled files.

### Example variables

```text
DJANGO_SECRET_KEY=
DJANGO_DEBUG=
DJANGO_ALLOWED_HOSTS=

DATABASE_URL=

CORS_ALLOWED_ORIGINS=
CSRF_TRUSTED_ORIGINS=

REDIS_URL=

EMAIL_HOST=
EMAIL_PORT=
EMAIL_HOST_USER=
EMAIL_HOST_PASSWORD=
EMAIL_USE_TLS=

SENTRY_DSN=
```

Only define variables that the project actually uses.

### `.env.example`

Keep `.env.example` synchronized with configuration:

```text
DJANGO_SECRET_KEY=replace-me
DJANGO_DEBUG=false
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

DATABASE_URL=postgresql://user:password@localhost:5432/app_db

CORS_ALLOWED_ORIGINS=http://localhost:3000
CSRF_TRUSTED_ORIGINS=http://localhost:3000
```

Do not put production credentials into this file.

---

## 1.4 Settings structure

Separate settings by environment when the project has materially different runtime behavior.

Recommended structure:

```text
config/
├── __init__.py
├── urls.py
├── asgi.py
├── wsgi.py
└── settings/
    ├── __init__.py
    ├── base.py
    ├── development.py
    ├── production.py
    └── test.py
```

### `base.py`

Contains shared configuration:

- installed applications;
- middleware;
- templates;
- database defaults/schema;
- internationalization;
- DRF;
- authentication defaults;
- pagination;
- logging base configuration;
- static/media configuration;
- shared security-safe defaults.

### `development.py`

Contains developer conveniences such as:

- `DEBUG=True`;
- local-only tooling;
- development email backend;
- permissive local settings where appropriate.

### `production.py`

Contains production-hardening:

- `DEBUG=False`;
- secure cookies;
- HTTPS-related settings;
- strict hosts;
- secure proxy handling;
- production logging;
- production email;
- production storage;
- production observability.

### `test.py`

Contains test-specific behavior such as:

- test database configuration;
- faster password hashing;
- deterministic test settings;
- disabled external integrations where appropriate.

Do not duplicate large settings blocks across files. Keep shared configuration in `base.py`.

---

## 1.5 Core Django settings

A baseline `INSTALLED_APPS` should contain only what the project needs:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "rest_framework",

    # Project applications go here.
]
```

Do not install applications that are not used.

### Middleware

Start with Django's standard middleware and add middleware only when its purpose is understood.

Typical baseline:

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

If an API uses cookie/session authentication, CSRF protection remains important.

If the project uses only non-cookie authentication for API requests, do not disable CSRF globally without understanding the browser/session implications.

### Templates

Even API-only projects normally keep the minimum Django template configuration required by installed framework components such as admin.

Do not remove framework configuration blindly to make the project "API only."

### Request limits

Use deployment-layer and application-layer limits for request size where needed.

At minimum, consider:

- upload size;
- JSON body size;
- reverse-proxy request limits;
- application-level validation.

---

## 1.6 Database configuration

PostgreSQL should be the production database.

Example conceptual configuration:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": ...,
        "USER": ...,
        "PASSWORD": ...,
        "HOST": ...,
        "PORT": ...,
        "CONN_MAX_AGE": 60,
    }
}
```

The exact connection management strategy must match deployment infrastructure.

### Database rules

- Database credentials come from configuration.
- Do not hard-code credentials.
- Do not use SQLite in production unless the product explicitly requires it.
- Use migrations for schema changes.
- Verify migrations in CI.
- Use PostgreSQL for tests of PostgreSQL-specific behavior.
- Configure connection lifetime deliberately.
- Monitor connection count in production.
- Do not open unnecessary database connections from background workers.

### Transactions

Django transaction behavior should be explicit.

Do not enable global `ATOMIC_REQUESTS` merely because it looks safer.

Use `transaction.atomic()` around operations that must commit or roll back as one unit.

Global request-wide transactions can create unnecessary locking and longer transaction lifetimes.

---

## 1.7 Django REST Framework configuration

Define a project-wide DRF baseline in settings.

Example:

```python
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
    ],

    "DEFAULT_PARSER_CLASSES": [
        "rest_framework.parsers.JSONParser",
    ],

    "DEFAULT_AUTHENTICATION_CLASSES": [
        # Configure the project's actual authentication mechanism here.
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],

    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",

    "DEFAULT_PAGINATION_CLASS": (
        "rest_framework.pagination.PageNumberPagination"
    ),

    "PAGE_SIZE": 25,

    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.OrderingFilter",
    ],

    "EXCEPTION_HANDLER": "config.exceptions.api_exception_handler",

    "DEFAULT_THROTTLE_CLASSES": [],
    "DEFAULT_THROTTLE_RATES": {},
}
```

The exact classes depend on installed packages and the project's authentication strategy.

### Important defaults

A production API should explicitly decide:

- authentication;
- permissions;
- renderer/parser types;
- pagination;
- filtering;
- ordering;
- exception handling;
- API schema generation;
- throttling/rate limiting.

Do not rely on implicit framework defaults for security-sensitive behavior.

### Global permission default

For most authenticated business APIs:

```python
"DEFAULT_PERMISSION_CLASSES": [
    "rest_framework.permissions.IsAuthenticated",
]
```

Public endpoints should opt out explicitly rather than making the entire API public by default.

For example:

```python
permission_classes = [AllowAny]
```

should be intentional and reviewable.

---

## 1.8 Authentication configuration

Authentication answers:

> Who is making the request?

Permission answers:

> What is that authenticated identity allowed to do?

These must remain separate.

### Authentication mechanism

Choose one primary API authentication strategy based on the client architecture.

Common options include:

- session authentication for browser-first applications;
- token authentication;
- JWT-based authentication;
- another established identity provider integration.

Do not configure several authentication mechanisms without a concrete use case.

### Security rule

Authentication credentials must never be accepted from:

- ordinary JSON body fields;
- URL query parameters;
- arbitrary custom fields.

Use the mechanism's documented transport, normally the `Authorization` header and/or secure cookies depending on the authentication design.

### Custom user model

If the project needs a custom user model, decide this **before the first production migration**.

Typical configuration:

```python
AUTH_USER_MODEL = "users.User"
```

Do not change `AUTH_USER_MODEL` casually after the project has accumulated migrations and production data.

### Passwords

Use Django's password hashing framework.

Do not implement custom password hashing.

Configure Django's password validators appropriate to the product.

---

## 1.9 CORS and CSRF

CORS and CSRF solve different problems.

### CORS

CORS controls which browser origins may call the API from a different origin.

Configure explicit allowed origins.

Prefer:

```python
CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
]
```

Avoid broad production configuration such as:

```python
CORS_ALLOW_ALL_ORIGINS = True
```

unless there is a documented reason.

Do not use `CORS_ALLOW_CREDENTIALS=True` unless credentialed cross-origin requests are actually required.

### CSRF

CSRF protects cookie-authenticated browser requests.

Configure trusted origins explicitly:

```python
CSRF_TRUSTED_ORIGINS = [
    "https://app.example.com",
]
```

Do not confuse:

```text
CORS_ALLOWED_ORIGINS
```

with:

```text
CSRF_TRUSTED_ORIGINS
```

They serve different security boundaries.

### Cookie authentication

When authentication uses cookies, review:

```python
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"

CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = False
CSRF_COOKIE_SAMESITE = "Lax"
```

The exact SameSite policy depends on the frontend deployment topology.

---

## 1.10 Security settings

Production settings must explicitly harden Django.

At minimum, review:

```python
DEBUG = False

SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"

CSRF_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = "Lax"

SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = "DENY"
SECURE_REFERRER_POLICY = "same-origin"
```

### HTTPS

If the application is served behind a reverse proxy, configure proxy headers correctly before enabling strict HTTPS behavior.

Typical settings may include:

```python
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

Only enable this when the proxy is trusted and correctly configured.

### HSTS

HSTS can be enabled in production:

```python
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = False
```

Do not enable preload casually. HSTS mistakes can make a domain difficult to recover from.

### Hosts

Production must use an explicit:

```python
ALLOWED_HOSTS = [...]
```

Do not use `["*"]` in production unless there is a documented infrastructure reason.

### Secrets

Never log or expose:

- passwords;
- access tokens;
- refresh tokens;
- API keys;
- database passwords;
- Django `SECRET_KEY`;
- authorization headers.

---

## 1.11 URLs and API base path

Keep the project URL configuration centralized.

Example:

```python
urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/v1/", include("api.urls")),
]
```

Keep API versioning consistent.

Do not mix:

```text
/api/
/api/v1/
/v1/api/
```

without a documented strategy.

### Trailing slashes

Choose one project-wide convention and keep it consistent.

The choice is less important than consistency across:

- URLs;
- frontend clients;
- documentation;
- tests;
- proxies.

---

## 1.12 Serialization and rendering defaults

For a JSON API, prefer JSON-only rendering unless another representation is explicitly required:

```python
DEFAULT_RENDERER_CLASSES = [
    "rest_framework.renderers.JSONRenderer",
]
```

This avoids exposing the DRF browsable API in production responses unnecessarily.

For production, the browsable API may be disabled while retaining it for development/testing when useful.

### Parsers

Use only required parsers.

For JSON-only APIs:

```python
DEFAULT_PARSER_CLASSES = [
    "rest_framework.parsers.JSONParser",
]
```

Add multipart parsing only for endpoints that actually receive file uploads.

Do not enable parsers globally without need.

---

## 1.13 Pagination, filtering, and ordering defaults

Collection endpoints must be bounded.

Set a project-wide default:

```python
DEFAULT_PAGINATION_CLASS = (
    "rest_framework.pagination.PageNumberPagination"
)

PAGE_SIZE = 25
```

The exact page size should be chosen from expected response sizes and client behavior.

### Filtering

Use explicit filter fields.

Do not allow clients to construct arbitrary ORM queries.

### Ordering

Whitelist ordering fields:

```python
ordering_fields = [
    "date_creation",
    "nom",
]
```

Avoid exposing sensitive or expensive database columns by default.

### Large datasets

For very large append-heavy collections, cursor pagination may be more appropriate.

Do not choose cursor pagination automatically; choose it based on access patterns.

---

## 1.14 API error handling

Use one global DRF exception handler.

The project should return a consistent error shape.

Example:

```json
{
  "success": false,
  "code": "VALIDATION_ERROR",
  "message": "Données invalides.",
  "errors": {
    "email": [
      "Adresse invalide."
    ]
  },
  "correlation_id": "..."
}
```

### Rules

- validation failures have stable error codes;
- authentication failures are distinguishable from authorization failures;
- not-found responses do not expose internal database details;
- unexpected errors return a generic response;
- stack traces stay server-side;
- response formats remain consistent.

Do not create different error envelopes for different endpoints.

### Exception handler

The handler should wrap/normalize DRF exceptions and application exceptions.

It should log unexpected failures with:

- correlation ID;
- request metadata;
- exception type;
- traceback;
- safe contextual fields.

Never log secrets or complete authorization headers.

---

## 1.15 API documentation

Use an OpenAPI schema generator such as `drf-spectacular` when API documentation is required.

Recommended baseline:

```python
REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": (
        "drf_spectacular.openapi.AutoSchema"
    ),
}
```

Configure:

```python
SPECTACULAR_SETTINGS = {
    "TITLE": "Project API",
    "DESCRIPTION": "REST API",
    "VERSION": "1.0.0",
}
```

The documentation must reflect the real API.

CI should detect schema-generation failures.

For security-sensitive endpoints, verify that documentation does not accidentally expose credentials, internal fields, or implementation details.

---

## 1.16 Static files and media

Separate static assets from user-uploaded media.

```text
STATIC_ROOT → collected application static files
MEDIA_ROOT  → uploaded user/application files
```

Do not mix them.

### Production

Prefer serving production static/media through:

- reverse proxy;
- object storage;
- CDN;
- another dedicated file-serving mechanism.

The Django application should not become an unnecessary file server under heavy traffic.

### Upload security

For uploaded files:

- validate size;
- validate content/type;
- do not trust the filename;
- generate safe storage names;
- authorize downloads;
- restrict access to the owning resource;
- define retention rules;
- never execute uploaded content.

---

## 1.17 Time zone and localization

Use timezone-aware datetimes.

Recommended:

```python
USE_TZ = True
```

Configure the project's actual timezone deliberately:

```python
TIME_ZONE = "Africa/Algiers"
```

Do not rely on the server's OS timezone.

Persist timestamps in UTC and convert for presentation when the product requires local time.

Set language explicitly:

```python
LANGUAGE_CODE = "fr-fr"
```

only when that is actually the product requirement.

Do not make a project language choice accidentally through the developer's machine locale.

---

## 1.18 Email

Configure email centrally.

Development should not accidentally send real messages.

Use a console/file backend locally when appropriate:

```python
EMAIL_BACKEND = (
    "django.core.mail.backends.console.EmailBackend"
)
```

Production should use a real provider configured through environment variables.

Configure:

- host;
- port;
- TLS/SSL;
- username;
- password;
- default sender;
- timeout.

Email sending should have explicit timeout and failure handling.

Do not block critical database transactions on slow email delivery when asynchronous delivery is available.

---

## 1.19 Cache and Redis

Caching is optional.

Redis should be added only when the project has a concrete need such as:

- caching;
- rate limiting;
- Celery broker;
- short-lived distributed state.

Example cache configuration:

```python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "...",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        },
    }
}
```

The project may use another supported backend.

### Cache rules

- PostgreSQL/database remains the source of truth.
- Cache keys must be deterministic.
- TTL must be intentional.
- Invalidation strategy must be understood.
- Do not cache highly mutable critical state without a consistency strategy.
- Do not introduce Redis solely because "enterprise applications use Redis".

---

## 1.20 Background jobs

Celery is optional.

Use background workers for:

- heavy processing;
- email delivery;
- large exports/imports;
- PDF generation;
- external synchronization;
- scheduled jobs;
- long-running reports.

Do not move trivial synchronous work to Celery just because a queue exists.

Tasks should have:

- explicit timeouts;
- bounded retries;
- safe retry behavior;
- observable failures;
- idempotency where duplicate execution is possible.

Never place an entire application's business workflow inside a Celery task. The task should invoke the application's normal operation.

---

## 1.21 Logging and observability

Configure structured or consistently formatted application logging from the beginning.

Baseline log categories should make it possible to distinguish:

```text
application
security
database
background jobs
external integrations
```

### Logging rules

Log enough context to investigate failures:

- timestamp;
- severity;
- logger;
- request/correlation ID;
- endpoint;
- user identifier when safe;
- operation;
- exception information.

Do not log:

- passwords;
- authorization headers;
- tokens;
- API keys;
- full request bodies when they may contain sensitive data;
- personal data unnecessarily.

### Correlation ID

Every API request should have a correlation ID.

Prefer accepting a trusted incoming correlation identifier or generating one server-side.

Propagate it to:

```text
HTTP request
 ↓
logs
 ↓
background job
 ↓
external request
```

Do not trust arbitrary client metadata as an authentication or authorization signal.

### Error tracking

Sentry or an equivalent service is optional but recommended for production applications where operational visibility matters.

---

## 1.22 Health and readiness endpoints

Provide operational endpoints suitable for deployment and monitoring.

Typical separation:

```text
/health/live
/health/ready
```

### Liveness

Answers:

> Is this process running?

It should remain lightweight.

### Readiness

Answers:

> Is this instance ready to receive traffic?

It may verify critical dependencies such as the database.

Do not make liveness depend on every external dependency.

Do not create health endpoints that expose credentials, internal topology, stack traces, or sensitive infrastructure details.

---

## 1.23 Development settings

Development settings should optimize for feedback speed without weakening production assumptions blindly.

Typical development choices:

```python
DEBUG = True
```

Use development-only tools such as the browsable API where useful.

Development may use:

- local PostgreSQL;
- console email backend;
- local file storage;
- permissive local CORS;
- debug logging.

Do not copy development security settings into production.

Do not use development credentials against production systems.

---

## 1.24 Production settings

Production MUST explicitly configure:

```python
DEBUG = False
```

and verify:

- secret key is externalized;
- allowed hosts are explicit;
- HTTPS is correctly configured;
- secure cookies are enabled when applicable;
- CSRF is correctly configured for cookie-authenticated clients;
- CORS is explicit;
- database credentials are externalized;
- error reporting is active;
- logging is configured;
- static/media serving is production-safe;
- allowed origins are explicit;
- request limits exist where necessary.

Before deployment, run:

```bash
python manage.py check --deploy
```

Deployment must fail or be blocked when critical security configuration is invalid.

---

## 1.25 Test settings

Tests should be fast where possible, but never at the cost of hiding important PostgreSQL behavior.

Use a dedicated test settings module when needed:

```text
config/settings/test.py
```

Typical test configuration may include:

- faster password hashing;
- disabled real email delivery;
- disabled external notifications;
- deterministic timezone/locale;
- dedicated test database;
- test-specific Celery behavior.

### Database

Use PostgreSQL for tests that verify:

- SQL semantics;
- constraints;
- transaction behavior;
- locking;
- concurrency;
- PostgreSQL-specific features.

### External services

Default tests should not call production external services.

Use:

- mocks;
- local test doubles;
- dedicated sandbox systems;
- recorded fixtures where appropriate.

---

## 1.26 ASGI / WSGI

Use the correct deployment interface for the runtime.

Django applications can expose:

```text
config/asgi.py
config/wsgi.py
```

ASGI is the preferred interface when the deployment requires asynchronous server capabilities.

WSGI remains valid for traditional synchronous deployment.

Do not add ASGI-specific complexity unless the runtime actually needs it.

---

## 1.27 Dependency and security checks

The project should have automated dependency and configuration checks.

At minimum, CI should run:

```bash
python manage.py check
python manage.py check --deploy
python manage.py makemigrations --check --dry-run
```

Add:

- formatter/linter;
- static/type checks where adopted;
- dependency vulnerability scanning;
- tests;
- migration checks.

Do not treat a vulnerability scanner as a replacement for secure engineering.

Keep dependencies reasonably current.

Apply security updates promptly.

---

## 1.28 Docker and deployment configuration

Containerization is optional, but if Docker is used, the image should follow predictable runtime principles.

Typical components:

```text
Dockerfile
.dockerignore
compose.yaml        # local/development if needed
```

### Container rules

- do not run the application as root when avoidable;
- do not bake secrets into images;
- keep images reproducible;
- separate build-time and runtime configuration;
- write only required temporary data to the container filesystem;
- send logs to stdout/stderr for platform collection;
- use health/readiness checks;
- keep the application process focused on one primary responsibility.

### Migrations

Do not hide complicated schema migrations inside application startup.

Migration execution should be an explicit deployment step.

---

## 1.29 CI baseline

CI should verify the project before merging.

Minimum pipeline:

```text
install dependencies
        ↓
lint / format check
        ↓
type/static checks if adopted
        ↓
Django system checks
        ↓
migration check
        ↓
tests
        ↓
security/dependency checks
```

For PostgreSQL-dependent projects, CI should provide PostgreSQL.

CI should not rely on a developer's local database.

### Recommended checks

```bash
python manage.py check
python manage.py makemigrations --check --dry-run
pytest
```

or the project's chosen test runner.

If OpenAPI documentation is part of the project contract, CI should also verify schema generation.

---

## 1.30 Initial project validation

Before feature development starts, the project must successfully pass the following baseline:

```bash
python manage.py check
python manage.py check --deploy
python manage.py makemigrations --check --dry-run
python manage.py migrate
```

Then verify:

- the API starts successfully;
- database connectivity works;
- authentication configuration behaves as intended;
- a protected endpoint rejects unauthenticated access;
- a public endpoint is public only when deliberately configured;
- CORS behavior matches the documented origins;
- CSRF behavior matches the authentication strategy;
- API error responses use the standard error contract;
- OpenAPI generation succeeds if enabled;
- logs are produced without secrets;
- health endpoints behave correctly;
- test suite passes;
- production settings fail safely when mandatory variables are missing.

Only after these checks pass should feature-level development begin.

---

# Chapter 2 — Strict Configuration Checklist

Every new Django/DRF project should pass all applicable items below.

## A. Project foundation

- [ ] Python version is explicitly chosen.
- [ ] Django version is explicitly chosen.
- [ ] DRF version is explicitly chosen.
- [ ] PostgreSQL is configured for production.
- [ ] Dependencies are reproducible/locked.
- [ ] `.gitignore` is correct.
- [ ] `.env.example` exists.
- [ ] No real secret is committed.
- [ ] README contains reproducible setup instructions.

## B. Settings

- [ ] Settings are centralized.
- [ ] Environment-specific settings are separated when useful.
- [ ] Shared settings are not duplicated unnecessarily.
- [ ] Required environment variables are validated.
- [ ] Missing mandatory production configuration fails fast.
- [ ] `DEBUG` is explicitly configured.
- [ ] `ALLOWED_HOSTS` is explicit.
- [ ] No business code reads environment variables directly.

## C. Database

- [ ] PostgreSQL is the production database.
- [ ] Credentials come from environment/secret management.
- [ ] Migrations are enabled.
- [ ] `makemigrations --check --dry-run` passes.
- [ ] PostgreSQL-specific behavior is tested against PostgreSQL.
- [ ] Connection lifetime is deliberately configured.
- [ ] Global `ATOMIC_REQUESTS` has not been enabled without justification.

## D. DRF

- [ ] Default renderer classes are explicitly configured.
- [ ] Default parser classes are explicitly configured.
- [ ] Authentication classes are explicitly configured.
- [ ] Default permission classes are explicitly configured.
- [ ] Public endpoints are explicit exceptions.
- [ ] Pagination is configured.
- [ ] Filtering is configured where needed.
- [ ] Ordering is configured where needed.
- [ ] Exception handler is configured.
- [ ] API schema generation is configured if used.
- [ ] Throttling/rate limiting has been considered.

## E. Authentication

- [ ] Authentication mechanism is explicitly chosen.
- [ ] Credentials are transported through the mechanism's secure channel.
- [ ] Credentials are not accepted through arbitrary JSON fields or query parameters.
- [ ] Custom user model decision is made before initial migrations.
- [ ] Passwords use Django's hashing framework.
- [ ] Password validation is configured.
- [ ] Authentication failures return appropriate HTTP status codes.

## F. Authorization

- [ ] Authentication and authorization are treated separately.
- [ ] Default permissions are restrictive enough.
- [ ] Object-level access is handled where needed.
- [ ] Client-supplied roles/permissions are never trusted.
- [ ] Authorization behavior is covered by tests.

## G. CORS / CSRF

- [ ] CORS origins are explicit.
- [ ] `CORS_ALLOW_ALL_ORIGINS` is not enabled casually in production.
- [ ] CSRF trusted origins are explicit.
- [ ] CORS and CSRF are configured independently.
- [ ] Cookie authentication has an explicit SameSite strategy.
- [ ] Secure cookies are enabled under HTTPS.
- [ ] Credentialed cross-origin requests are enabled only when actually needed.

## H. Security

- [ ] `DEBUG=False` in production.
- [ ] Production `SECRET_KEY` is externalized.
- [ ] `ALLOWED_HOSTS` is explicit.
- [ ] HTTPS redirect/proxy configuration has been verified where applicable.
- [ ] Security middleware is enabled.
- [ ] HSTS is reviewed before activation.
- [ ] Clickjacking protection is configured.
- [ ] Content-type sniffing protection is configured.
- [ ] Referrer policy is configured appropriately.
- [ ] Secrets/tokens are not logged.
- [ ] File uploads are validated.
- [ ] Request/upload size limits are considered.

## I. API contract

- [ ] API base path/versioning strategy is consistent.
- [ ] Trailing-slash policy is consistent.
- [ ] JSON response format is consistent.
- [ ] Error response format is consistent.
- [ ] Validation errors are structured.
- [ ] Unexpected errors do not leak internals.
- [ ] Correlation IDs are supported.
- [ ] OpenAPI documentation matches implementation.

## J. Pagination / filtering / ordering

- [ ] Collection endpoints are bounded.
- [ ] Default page size is configured.
- [ ] Filter fields are explicitly defined.
- [ ] Ordering fields are explicitly defined.
- [ ] Sensitive/expensive fields are not exposed for arbitrary ordering.
- [ ] Large datasets use an appropriate pagination strategy.

## K. Static/media

- [ ] `STATIC_ROOT` is configured for production.
- [ ] `MEDIA_ROOT` is separated from static files.
- [ ] Media access is authorized.
- [ ] Upload size is limited.
- [ ] Upload content/type is validated.
- [ ] User filenames are not trusted.
- [ ] Production file storage strategy is explicit.
- [ ] Uploaded files cannot be executed as application code.

## L. Time and localization

- [ ] `USE_TZ=True`.
- [ ] Project timezone is explicit.
- [ ] UTC persistence behavior is understood.
- [ ] Date/time serialization is tested.
- [ ] Language/locale is explicitly selected when required.

## M. Email

- [ ] Development cannot accidentally send production email.
- [ ] Production SMTP/API credentials are externalized.
- [ ] Email timeouts are configured.
- [ ] Default sender is configured.
- [ ] Critical request transactions do not unnecessarily wait for slow email delivery.

## N. Cache / Redis

- [ ] Redis is used only when justified.
- [ ] Cache is not the source of truth.
- [ ] Cache keys are deterministic.
- [ ] TTL is intentional.
- [ ] Invalidation strategy is understood.
- [ ] Sensitive mutable state is not cached without a consistency strategy.

## O. Background jobs

- [ ] Celery is used only when asynchronous execution is justified.
- [ ] Tasks have explicit retry behavior.
- [ ] Tasks have timeouts.
- [ ] Retry safety is understood.
- [ ] Duplicate execution is handled where necessary.
- [ ] Worker failures are observable.
- [ ] Scheduled tasks are explicitly configured.

## P. Logging / observability

- [ ] Logging is configured globally.
- [ ] Logs include useful context.
- [ ] Correlation ID is propagated.
- [ ] Secrets are never logged.
- [ ] Unexpected exceptions are tracked.
- [ ] Production error tracking is configured when required.
- [ ] Application, security, and integration failures can be distinguished.

## Q. Health / deployment

- [ ] Liveness endpoint exists where required.
- [ ] Readiness endpoint exists where required.
- [ ] Liveness does not depend on every external service.
- [ ] Readiness checks critical dependencies only.
- [ ] Health endpoints expose no sensitive information.
- [ ] ASGI/WSGI entrypoint is defined.
- [ ] Deployment migration step is explicit.
- [ ] Application startup does not depend on hidden migration execution.

## R. Testing

- [ ] Test settings exist where useful.
- [ ] Tests do not contact real production services.
- [ ] PostgreSQL is used for DB-specific tests.
- [ ] Authentication is tested.
- [ ] Authorization is tested.
- [ ] CORS/CSRF behavior is tested where relevant.
- [ ] API error contracts are tested.
- [ ] OpenAPI generation is tested if contractual.
- [ ] Migration checks pass.

## S. CI

- [ ] Dependencies install reproducibly.
- [ ] Lint/format checks pass.
- [ ] Static/type checks pass where adopted.
- [ ] `manage.py check` passes.
- [ ] `manage.py check --deploy` passes for production configuration.
- [ ] `makemigrations --check --dry-run` passes.
- [ ] Tests run against PostgreSQL where required.
- [ ] Dependency/security checks run.
- [ ] CI does not depend on developer-local services.

## T. Final readiness review

- [ ] A new developer can create the environment from documented steps.
- [ ] A new developer can run the project without manually editing source code.
- [ ] Missing production secrets cause a clear configuration failure.
- [ ] Protected API endpoints are protected by default.
- [ ] Public endpoints are deliberate and explicit.
- [ ] Database migrations are reproducible.
- [ ] Production security checks pass.
- [ ] API documentation can be generated.
- [ ] Logs can be correlated across a request.
- [ ] Health/readiness checks work in the deployment environment.
- [ ] Tests pass from a clean environment.

---

# Configuration Principle

The first guide should answer only:

> **Is the Django/DRF project correctly prepared and configured to build features safely?**

It should establish:

```text
Python
 ↓
Dependencies
 ↓
Environment / Secrets
 ↓
Django Settings
 ↓
PostgreSQL
 ↓
Django REST Framework
 ↓
Authentication / Authorization defaults
 ↓
CORS / CSRF / Security
 ↓
API Contract / Errors / OpenAPI
 ↓
Static / Media / Email
 ↓
Cache / Workers when justified
 ↓
Logging / Health / Deployment
 ↓
Tests / CI
```

It should **not** decide:

- how business modules are organized;
- how use cases are structured;
- where business rules live;
- whether a module uses a particular internal architecture;
- how business entities interact;
- how a specific ERP or SaaS domain is modeled.

Those decisions belong to the development/architecture guide that follows.
