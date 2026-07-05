# Full-Stack App — Deployment Plan v6 (Server-Hosted Frontend Variant)

Django backend (server) + Next.js frontend (**also server**) + Electron thin client (client PCs), LAN-deployed, Nuitka-compiled backend, licensed.

Backend and frontend are now **two services on the same server machine**, plus a separate, much lighter Electron shell shipped to every workstation. Each client PC no longer carries its own copy of the frontend build — it opens a small native window that loads the frontend live from the server over the LAN, the same way a browser would, but wrapped in a controlled, licensed, native shell instead of a bare browser tab.

> **This is a variant of Deployment Plan v5**, for clients whose local server is powerful enough to run the frontend as well as the backend, instead of shipping a full Next.js build to every PC. Everywhere you see `app-server`, `app-client`, `AppClient`, `AppServer`, or `com.yourcompany.appclient`, swap in your actual project's name — same reusable-template note as v5.

---

## What changed from v5

**The core shift:** the frontend stops being something every client PC builds and runs for itself, and becomes a third thing the server runs, alongside the backend. Electron stops being a container *for* the frontend and becomes a thin, licensed *window onto* it.

**Reordered / re-homed (one real ordering trap avoided, not just cosmetic):**
- The Nginx block that reverse-proxies the frontend is added in **Phase 8**, not Phase 2 — even though Phase 2 is where Nginx first gets configured (for `/static/` and `/media/`). Wiring a `proxy_pass` to a frontend build that doesn't exist yet is the exact same mistake v5 already caught once with Nuitka and systemd ("don't point a service config at a binary that isn't built yet"). Here it's one level up the stack: don't point a reverse-proxy `location` block at a frontend that isn't built yet.
- The frontend gets its **own OS service** (systemd/NSSM), following the same pattern as the backend's Phase 6 — but it lives inside Phase 8, because that's the earliest point in the plan where the frontend build actually exists. Phase 6 stays backend-only.

**Simplified:**
- The original problem that forced Electron into a runtime-config pattern — "`NEXT_PUBLIC_*` is baked in at build time, and every client machine might need a different value" — **no longer applies to the frontend itself**. The frontend is now built exactly once, for exactly one target (the server it runs on), so it can use relative paths (`/api/...`) and never needs to know a LAN IP at build time or runtime.
- CORS becomes largely unnecessary for browser-side calls once frontend and backend share one origin behind Nginx (Task 8.4). It's kept as a defense-in-depth fallback, not the load-bearing mechanism it was in v5.
- The client installer gets dramatically smaller: no Next.js build, no `app.asar` full of renderer code — just the Electron shell and one fallback HTML page.

**The runtime-config problem doesn't disappear — it moves:**
- v5's Electron needed to know the backend's LAN address (`API_URL`) because that's the one thing that varies per deployment and isn't known until install time.
- This variant's Electron needs to know the **server's** LAN address (`SERVER_URL`) for exactly the same reason — it's now loading the whole app UI from the network instead of the API alone. Same mechanism (`config.env` in `userData`, editable post-build), same reasoning, one variable instead of two.

**Added (new failure mode that v5 didn't have):**
- Electron now loads *remote* content instead of a *bundled local* file. That introduces two things v5's Electron never needed:
  1. A navigation guard (`will-navigate` / `setWindowOpenHandler`) so the loaded page can never carry the shell to some other origin.
  2. A `did-fail-load` fallback, for the case where the license check passes but the frontend page itself fails to load a moment later (frontend service down, Nginx down, transient LAN blip) — a failure mode that's meaningfully different from "license invalid," and v5's model had no equivalent for it.
- Content-Security-Policy for the app UI moves from "a `<meta>` tag Electron ships inside its own bundled HTML" to "a real HTTP response header the server sends," since Electron doesn't own the page's markup anymore.

**Unaffected — kept exactly as in v5:**
- Backend Django app, its `.env`/logging/licensing/Nuitka-compilation/service-wrapping/backup phases (Phases 2–7). The backend doesn't know or care that the frontend used to live on client PCs and now lives next to it.
- The licensing model itself (one license bound to the server machine; HWID hand-off; JWT/RS256 tokens; `/api/license/status/`).
- The overall two-tree separation (server artifacts vs. client artifacts), the "do permissions now, not at the end" discipline, and the phase-by-phase STOP-checkpoint structure.

---

## Index

1. [Phase 1 — Two Project Structures (Server vs. Client)](#phase-1)
2. [Phase 2 — Make the Backend Production Ready](#phase-2)
3. [Phase 3 — Logging](#phase-3)
4. [Phase 4 — Backend Licensing](#phase-4)
5. [Phase 5 — Compile Backend With Nuitka](#phase-5)
6. [Phase 6 — Windows Service / Linux Service (Backend)](#phase-6)
7. [Phase 7 — Backup Strategy](#phase-7)
8. [Phase 8 — Frontend Hosted on the Server + Electron Thin Client](#phase-8)
9. [Phase 9 — Frontend Licensing Gate in Electron](#phase-9)
10. [Phase 10 — Two Installers (Server + Client)](#phase-10)
11. [Phase 11 — Security Hardening (Audit Pass)](#phase-11)
12. [Milestone Order](#milestone-order)
13. [Future Feature — License Renewal Endpoint](#future-feature)

---

<a id="phase-1"></a>
# Phase 1 — Two Project Structures (Server vs. Client)

Goal: separate the deployment artifacts for the server (backend **and now frontend**) from the client PCs (Electron shell only) from day one. They will never ship together.

## Task 1.1 — Backend + Frontend (Server) Deployment Layout

```text
app-server/
├── backend/            # compiled backend binary (Nuitka output)
├── frontend/           # Next.js production build — runs on the server now (NEW)
├── config/             # .env, frontend.env, license.lic, public_key.pem
├── data/                # sqlite database
├── media/               # uploaded files
├── logs/                 # backend logs, frontend logs, service logs
├── backups/               # zip archives produced by backup.sh / backup.bat
└── scripts/                # startup / backup / hwid scripts
```

```text
backend/  -> executable files (backend.exe / backend.bin)
frontend/ -> Next.js standalone server output (server.js + .next/ + public/)  [NEW]
config/   -> .env, frontend.env, license.lic, public_key.pem
data/     -> db.sqlite3
media/    -> uploaded files
logs/     -> backend logs, frontend-stdout.log, frontend-stderr.log
backups/  -> backup_YYYY-MM-DD_HHMMSS.zip
scripts/  -> backup.sh / backup.bat / get_machine_id.py
```

The `frontend/` tree doesn't exist yet at this point in the plan — it's built in Phase 8. It's listed here so the final directory shape is clear from the start, the same way `backend/` was listed in Phase 1 before Nuitka produced anything to put in it.

## Task 1.2 — Frontend (Client) Deployment Layout — Now Just an Electron Shell

```text
app-client/
├── app/                          # Electron shell only — no bundled application UI
│   ├── AppClient.exe             # the Electron launcher
│   ├── resources/
│   │   ├── app.asar              # Electron main/preload/fallback-HTML ONLY — a few KB
│   │   └── default.env           # template copied to userData on first run
│   └── ... (chrome runtime, etc.)
└── (per-user runtime data — NOT shipped)
    └── %APPDATA%/AppClient/
        ├── config.env            # <-- USER-EDITABLE, now holds SERVER_URL
        └── logs\
```

This is the biggest structural difference from v5: there is no packaged Next.js build inside `app.asar` at all. The Electron shell's only job is to (1) gate on the license, (2) point a native window at the server, and (3) show a local fallback screen if it can't. The actual application — every screen, every component — is fetched live from `SERVER_URL` each time the shell opens. Fixing a frontend bug means redeploying `app-server/frontend/` once; it does not mean rebuilding or redistributing anything to client PCs.

The user-editable file is still **`config.env`**, stored in `app.getPath('userData')` on each client PC — see Phase 8 for the mechanism, carried over unchanged from v5.

## Task 1.3 — Move SQLite Outside Backend Code

```text
backend/db.sqlite3  ->  app-server/data/db.sqlite3
```

Unchanged from v5.

## Task 1.4 — Move Media Outside Backend Code

```text
backend/media/  ->  app-server/media/
```

Unchanged from v5.

## Task 1.5 — Move Backend `.env` Outside Code

```text
backend/.env  ->  app-server/config/.env
```

```env
SECRET_KEY=change-me
DEBUG=False
DATABASE_URL=sqlite:///data/db.sqlite3

# server
PORT=8000
ALLOWED_HOSTS=192.168.1.100,localhost,127.0.0.1

# CORS — only load-bearing if you skip the unified Nginx origin in Phase 8 Task 8.4.
# With the unified origin, the frontend calls the API same-origin through Nginx, and
# this becomes defense-in-depth rather than the thing actually preventing cross-origin calls.
CORS_ALLOWED_ORIGINS=http://192.168.1.100
```

`config/license.lic` and `config/public_key.pem` are still fixed, hardcoded filenames under `CONFIG_DIR` — no env var needed for them. Same as v5.

**Path resolution:** unchanged from v5 — resolve every backend path from one explicit root, not `__file__`, since Nuitka `--onefile` extracts to a temp folder at runtime.

```python
# settings.py / run_server.py
import os
DEPLOY_ROOT = os.environ.get("DEPLOY_ROOT") or os.getcwd()
CONFIG_DIR  = os.path.join(DEPLOY_ROOT, "config")
DATA_DIR    = os.path.join(DEPLOY_ROOT, "data")
MEDIA_DIR   = os.path.join(DEPLOY_ROOT, "media")
LOG_DIR     = os.path.join(DEPLOY_ROOT, "logs")
BACKUP_DIR  = os.path.join(DEPLOY_ROOT, "backups")
```

```python
from dotenv import load_dotenv
load_dotenv(os.path.join(CONFIG_DIR, ".env"))
```

**Permissions (do this now, not at the end):**

```bash
mkdir -p app-server/logs app-server/backups
chmod 600 app-server/config/.env
chown app-svc:app-svc -R app-server/config
```

Nothing frontend-related gets created, configured, or permissioned yet — `app-server/frontend/` stays a pure placeholder in the directory tree above until Phase 8, after every backend phase (2 through 7) is finished, compiled, serviced, and backed up. That ordering is deliberate and holds for the rest of this plan: **no frontend task starts before the backend track reaches its own final STOP.**

---

# STOP

```text
✓ two separate project trees exist (app-server/, app-client/)
✓ backend sqlite/media/.env outside code, chmod 600, owned by service account
✓ DEPLOY_ROOT resolves correctly when launched from a different working directory
✓ client userData path identified as the home of the runtime config.env
✓ app-client's planned layout has no bundled application UI — confirmed shell-only
✓ nothing under app-server/frontend/ has been created or configured yet — that's Phase 8, after Phase 7
```

---

<a id="phase-2"></a>
# Phase 2 — Make the Backend Production Ready

Unchanged from v5, with one deferred item called out explicitly below (Task 2.3b).

## Task 2.1 — Install Dependencies

```bash
pip install waitress python-dotenv whitenoise django-cors-headers
```

`django-cors-headers` stays installed even with the unified-origin approach in Phase 8 — keep it as defense-in-depth (Task 1.5), and it's the only thing standing between the Electron client and the API if you choose *not* to unify origins (see Task 2.3b).

## Task 2.2 — Static Files (Whitenoise)

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...rest of your middleware
]

STATIC_URL = "/static/"
STATIC_ROOT = os.path.join(DEPLOY_ROOT, "staticfiles")
STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
```

```bash
python manage.py collectstatic --noinput
```

## Task 2.3 — Media Files, and Where Nginx Fits (Backend Half Only)

**Option A — Nginx reverse proxy (recommended for LAN deployments):**

```nginx
# /etc/nginx/sites-available/app-server
server {
    listen 80;
    server_name app-server.local;

    location /static/ {
        alias /opt/app/app-server/staticfiles/;
    }

    location /media/ {
        alias /opt/app/app-server/media/;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/app-server /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

**Option B — Authenticated Django view** streaming files from `MEDIA_DIR`, same tradeoff as v5.

### Task 2.3b — The Frontend `location /` Block Comes Later, On Purpose

This same config file gets one more `location` block in **Phase 8, Task 8.4**, once the Next.js frontend actually exists to proxy to. Adding it now would mean reloading Nginx with a `proxy_pass` pointing at a port nothing is listening on yet — same class of mistake v5 fixed once already (compiling the backend before writing the systemd unit that points at it). This phase intentionally stops at static/media/API only.

## Task 2.4 — CORS and Allowed Hosts, Read From `.env`

```python
# settings.py
import os

ALLOWED_HOSTS = [h.strip() for h in os.environ.get("ALLOWED_HOSTS", "").split(",") if h.strip()]

CORS_ALLOWED_ORIGINS = [
    o.strip() for o in os.environ.get("CORS_ALLOWED_ORIGINS", "").split(",") if o.strip()
]
CORS_ALLOW_CREDENTIALS = True
```

**How this plays out once Phase 8 is done:** if you go with the unified Nginx origin (Task 8.4, recommended), every request the browser makes to `/api/...` is same-origin as far as the browser is concerned — it never triggers a CORS preflight at all, because it's not cross-origin. `CORS_ALLOWED_ORIGINS` still matters as a second line of defense, since Django's own port (8000) is still technically reachable from the LAN unless you also close it off in Phase 11. If you skip the unified origin and keep the frontend on its own port with no reverse proxy in front of it, CORS is back to being load-bearing exactly as it was in v5 — lock `CORS_ALLOWED_ORIGINS` to that port's origin.

## Task 2.5 — Create Startup File

```python
# run_server.py
import os
from dotenv import load_dotenv

DEPLOY_ROOT = os.environ.get("DEPLOY_ROOT") or os.getcwd()
load_dotenv(os.path.join(DEPLOY_ROOT, "config", ".env"))

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")

from waitress import serve
from config.wsgi import application

port = int(os.environ.get("PORT", 8000))
serve(application, host="0.0.0.0", port=port)
```

Unchanged from v5. `host="0.0.0.0"` is fine here because Phase 11 will bind this to `127.0.0.1` once Nginx is fronting it — see Task 2.6 and Phase 11.

## Task 2.6 — Open the Firewall

```bash
# Linux — ufw
sudo ufw allow 8000/tcp

# Linux — firewalld
sudo firewall-cmd --add-port=8000/tcp --permanent
sudo firewall-cmd --reload
```

```powershell
# Windows
netsh advfirewall firewall add rule name="App Server API" dir=in action=allow protocol=TCP localport=8000
```

This opens port 8000 for now, so you can test the API directly from another PC in this phase, before Nginx and the frontend exist. **This gets reversed in Phase 11** — once the unified origin is live, 8000 should be closed to the LAN and only Nginx's port (80/443) stays open.

## Task 2.7 — Test From Another PC

```bash
python run_server.py
```

From a second machine, hit `http://SERVER_IP:PORT`. Verify the API works from another machine, not just `localhost`.

---

# STOP

```text
✓ another PC can reach the API on the port defined in .env
✓ CORS headers present on responses
✓ ALLOWED_HOSTS and CORS_ALLOWED_ORIGINS are read from .env, not hardcoded
✓ collectstatic has run and static assets load correctly
✓ firewall rule for PORT confirmed on the actual OS you're deploying to
✓ Nginx serves /static/ and /media/ and proxies /api/ and /admin/ — no frontend block yet, by design
```

---

<a id="phase-3"></a>
# Phase 3 — Logging

Unchanged from v5 — this phase governs **backend (Django) logging only**. The Next.js frontend's process logging is handled separately by its own service wrapper in Phase 8, writing to `logs/frontend-stdout.log` / `logs/frontend-stderr.log` alongside the files below.

```text
app-server/logs/
```

```python
# settings.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{asctime} {levelname} {module} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "app_file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": os.path.join(LOG_DIR, "application.log"),
            "maxBytes": 5 * 1024 * 1024,
            "backupCount": 5,
            "formatter": "verbose",
        },
        "error_file": {
            "level": "ERROR",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": os.path.join(LOG_DIR, "errors.log"),
            "maxBytes": 5 * 1024 * 1024,
            "backupCount": 5,
            "formatter": "verbose",
        },
        "license_file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": os.path.join(LOG_DIR, "license.log"),
            "maxBytes": 2 * 1024 * 1024,
            "backupCount": 3,
            "formatter": "verbose",
        },
    },
    "loggers": {
        "django": {
            "handlers": ["app_file", "error_file"],
            "level": "INFO",
            "propagate": True,
        },
        "license": {
            "handlers": ["license_file"],
            "level": "INFO",
            "propagate": False,
        },
    },
}
```

---

# STOP

```text
✓ application.log, errors.log, license.log all write to app-server/logs/
✓ log rotation confirmed working (maxBytes/backupCount honored)
```

---

<a id="phase-4"></a>
# Phase 4 — Backend Licensing

Unchanged from v5. The licensing model still binds the license to the **server machine** only — and that server machine now happens to run the frontend too, so one license continues to cover the whole deployment, not per-client licensing.

## Task 4.1 — Design the License Token

```bash
pip install pyjwt cryptography
```

```json
{
  "client": "Customer / Deployment Name",
  "license_id": "uuid",
  "hwid": "sha256 of machine fingerprint",
  "features": ["billing", "reports"],
  "max_clients": 10,
  "mode": "offline",
  "server_url": "https://license.yourcompany.com/api/validate",
  "grace_days": 14,
  "iat": 1751328000,
  "exp": 1782864000
}
```

`max_clients` is still informational — actual concurrent-client enforcement (Electron windows hitting the same backend) is a backend concern.

## Task 4.2 — Generate a Signing Keypair (you keep this, once)

```bash
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

- `private_key.pem` — stays on your machine, never shipped.
- `public_key.pem` — ships with the backend, goes in `app-server/config/public_key.pem`.

## Task 4.3 — Machine Fingerprint Script

```python
# scripts/get_machine_id.py
import hashlib, platform, uuid

def machine_id():
    raw = f"{uuid.getnode()}-{platform.node()}-{platform.system()}"
    return hashlib.sha256(raw.encode()).hexdigest()

if __name__ == "__main__":
    print(machine_id())
```

### Task 4.3b — The HWID Hand-off Workflow

Unchanged from v5:

1. During server install, run `python scripts/get_machine_id.py`.
2. Whoever is on-site copies the printed hash and sends it to you.
3. You paste that hash into `issue_license.py`'s `hwid` field and run it (Task 4.4).
4. You send back the resulting `license.lic` for them to place in `config/license.lic`.
5. If the server's hardware changes later, repeat this loop (or use the Future Feature renewal endpoint).

## Task 4.4 — Issue a License (your side, offline tool, not shipped)

```python
# scripts/issue_license.py (keep on your machine, never distribute)
import jwt, time

payload = {
    "client": "Customer / Deployment Name",
    "license_id": "deploy-001",
    "hwid": "<paste from get_machine_id.py>",
    "features": ["billing", "reports"],
    "max_clients": 10,
    "mode": "offline",
    "server_url": "https://license.yourcompany.com/api/validate",
    "grace_days": 14,
    "iat": int(time.time()),
    "exp": int(time.time()) + 60 * 60 * 24 * 365,
}

with open("private_key.pem") as f:
    private_key = f.read()

token = jwt.encode(payload, private_key, algorithm="RS256")
with open("license.lic", "w") as f:
    f.write(token)
```

Send the client `license.lic` → they drop it in `app-server/config/license.lic`, `chmod 600`, owned by the service account.

## Task 4.5 — Backend Validation Module

```python
# license_check.py
import jwt, hashlib, platform, uuid, os, logging

logger = logging.getLogger("license")

def current_hwid():
    raw = f"{uuid.getnode()}-{platform.node()}-{platform.system()}"
    return hashlib.sha256(raw.encode()).hexdigest()

def get_public_key(config_dir):
    with open(os.path.join(config_dir, "public_key.pem")) as f:
        return f.read()

def validate_license_token(token, public_key):
    return jwt.decode(token, public_key, algorithms=["RS256"])

def validate_license(config_dir):
    public_key = get_public_key(config_dir)
    try:
        with open(os.path.join(config_dir, "license.lic")) as f:
            token = f.read().strip()
    except FileNotFoundError:
        logger.warning("License check failed: missing license.lic")
        raise ValueError("missing")

    try:
        payload = validate_license_token(token, public_key)
    except jwt.ExpiredSignatureError:
        logger.warning("License check failed: expired")
        raise ValueError("expired")
    except jwt.InvalidSignatureError:
        logger.warning("License check failed: tampered")
        raise ValueError("tampered")

    if payload["hwid"] != current_hwid():
        logger.warning("License check failed: hwid_mismatch")
        raise ValueError("hwid_mismatch")

    payload.setdefault("mode", "offline")
    payload.setdefault("grace_days", 14)
    logger.info(f"License check passed for {payload['client']} (exp={payload['exp']})")
    return payload
```

Wire this into Django as a startup check + a lightweight cached middleware, and expose `/api/license/status/`:

```json
{ "valid": true, "client": "...", "exp": 1782864000, "features": ["billing","reports"] }
```

or

```json
{ "valid": false, "reason": "expired" }
```

This endpoint is still what the Electron client calls on boot (Phase 9) — the only change from v5 is that it's now reached through the unified Nginx origin at `/api/license/status/` instead of directly at `API_URL`.

## Task 4.6 — Optional Online Mode

Unchanged from v5 — `mode: "online"` triggers a periodic phone-home to `server_url`, using `grace_days` to tolerate flaky connections.

## Task 4.7 — Test Backend Licensing

```text
✓ valid license          → app runs normally
✓ missing license file    → clear error, logged to license.log
✓ expired license         → clear error, logged
✓ license for wrong hwid  → clear error, logged
✓ tampered license file   → rejected (signature check fails), logged
✓ (online mode) revoked license → blocked after grace period
✓ (online mode) offline for < grace period → still runs
✓ /api/license/status/ returns correct JSON for each state
```

---

# STOP

```text
✓ backend refuses to run without a valid, matching license
✓ /api/license/status/ exposes a clean JSON status the Electron client can consume
✓ every failure path writes a line to logs/license.log
```

---

<a id="phase-5"></a>
# Phase 5 — Compile Backend With Nuitka

Unchanged from v5. This phase compiles the **Python/Django backend only** — the Next.js frontend is a Node app, not a Python one, and doesn't go through Nuitka; it gets its own build step in Phase 8.

```bash
pip install nuitka
```

```bash
python -m nuitka \
    --standalone \
    --onefile \
    --include-package=<your_django_app_1> \
    --include-package=<your_django_app_2> \
    --include-package-data=django \
    --enable-plugin=anti-bloat \
    --enable-plugin=data-files \
    run_server.py
```

Notes (unchanged from v5):
- `--include-package=` for **each Django app** is mandatory.
- `data-files` pulls in Django admin templates/static.
- If uploaded images go through Pillow: `--include-package=PIL`.
- **Compile on the same OS you're deploying to** — Nuitka does not cross-compile.
- **Windows toolchain:** MSVC or MinGW64 required; `--assume-yes-for-downloads` or `--msvc=latest`.
- Build inside a clean virtualenv matching the exact Python version you'll run in production.

## Test the Compiled Binary

```text
config/.env
config/license.lic
config/public_key.pem
data/db.sqlite3
media/
```

Re-run the entire Task 4.7 license lifecycle test against the compiled binary.

---

# STOP — Backend binary ready

```text
✓ compiled backend binary runs standalone from a plain shell (no service wrapper yet)
✓ reads config/data/media from outside the binary
✓ full license lifecycle (Task 4.7) re-tested against the compiled binary
✓ logging still writes correctly from the compiled binary
```

---

<a id="phase-6"></a>
# Phase 6 — Windows Service / Linux Service (Backend)

Unchanged from v5. This phase wraps the **compiled backend binary** from Phase 5 only. The frontend's Node process gets its own service unit in **Phase 8**, once its build exists — same reasoning v5 already applied to Nuitka-before-systemd, just one level up the stack: don't wire a service to something that isn't built yet.

## Linux (systemd)

```ini
# /etc/systemd/system/app.service
[Unit]
Description=App Server Backend
After=network.target

[Service]
Type=simple
User=app-svc
Group=app-svc
WorkingDirectory=/opt/app/app-server
Environment=DEPLOY_ROOT=/opt/app/app-server
ExecStart=/opt/app/app-server/backend/backend.bin
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable app.service
sudo systemctl start app.service
sudo systemctl status app.service
```

## Windows (NSSM)

```powershell
nssm install AppServer "C:\AppServer\backend\backend.exe"
nssm set AppServer AppDirectory "C:\AppServer"
nssm set AppServer AppEnvironmentExtra DEPLOY_ROOT=C:\AppServer
nssm set AppServer Start SERVICE_AUTO_START
nssm set AppServer AppStdout C:\AppServer\logs\service-stdout.log
nssm set AppServer AppStderr C:\AppServer\logs\service-stderr.log
nssm set AppServer ObjectName .\app-svc <service-account-password>
nssm start AppServer
```

Run under a low-privilege dedicated account (`app-svc`), not `LocalSystem` or an admin account.

## Verify

```bash
# Linux
sudo reboot
systemctl status app.service
curl http://localhost:8000/api/license/status/
```

```powershell
# Windows
shutdown /r /t 0
nssm status AppServer
Invoke-RestMethod http://localhost:8000/api/license/status/
```

---

# STOP

```text
✓ server reboot → backend service starts automatically → license still validates correctly
✓ service runs under a dedicated low-privilege account, not admin/root/LocalSystem
✓ stdout/stderr from the service land somewhere you can read
```

---

<a id="phase-7"></a>
# Phase 7 — Backup Strategy

Unchanged from v5, with `frontend.env` added to the backup list (small, config-like, same as `.env`) and `frontend/` explicitly added to the "never backup" list (it's a build artifact, same treatment as `backend/`).

Backup ONLY:

```text
app-server/config/   (includes .env, frontend.env, AND license.lic)
app-server/data/
app-server/media/
```

Never backup:

```text
app-server/backend/    # regenerate from source + Nuitka, don't archive binaries
app-server/frontend/   # regenerate from source via `npm run build`, don't archive build output
app-client/             # client PCs aren't backed up from the server side
```

## `scripts/backup.sh` (Linux)

```bash
#!/usr/bin/env bash
set -euo pipefail

DEPLOY_ROOT="/opt/app/app-server"
BACKUP_DIR="$DEPLOY_ROOT/backups"
DATE=$(date +%Y-%m-%d_%H%M%S)
ARCHIVE="$BACKUP_DIR/backup_${DATE}.zip"

mkdir -p "$BACKUP_DIR"

zip -r -e "$ARCHIVE" \
    "$DEPLOY_ROOT/config" \
    "$DEPLOY_ROOT/data" \
    "$DEPLOY_ROOT/media" \
    -x "*.pyc"

chmod 600 "$ARCHIVE"
echo "Backup created at $ARCHIVE"

find "$BACKUP_DIR" -name "backup_*.zip" -mtime +30 -delete
```

```bash
chmod +x scripts/backup.sh
```

## `scripts/backup.bat` (Windows)

```bat
@echo off
setlocal enabledelayedexpansion

set DEPLOY_ROOT=C:\AppServer
set BACKUP_DIR=%DEPLOY_ROOT%\backups

if not exist "%BACKUP_DIR%" mkdir "%BACKUP_DIR%"

for /f %%i in ('powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HHmmss"') do set STAMP=%%i

powershell -NoProfile -Command ^
  "Compress-Archive -Path '%DEPLOY_ROOT%\config','%DEPLOY_ROOT%\data','%DEPLOY_ROOT%\media' -DestinationPath '%BACKUP_DIR%\backup_%STAMP%.zip' -Force"

echo Backup created: %BACKUP_DIR%\backup_%STAMP%.zip

forfiles /p "%BACKUP_DIR%" /m backup_*.zip /d -30 /c "cmd /c del @path" 2>nul
```

Password-protection notes unchanged from v5 (`zip -e` for cron, or 7-Zip for Windows if `Compress-Archive`'s lack of encryption is a concern).

## Scheduling

```bash
0 2 * * * /opt/app/app-server/scripts/backup.sh >> /opt/app/app-server/logs/backup.log 2>&1
```

```powershell
schtasks /create /tn "AppServerBackup" /tr "C:\AppServer\scripts\backup.bat" /sc daily /st 02:00
```

The archive contains `SECRET_KEY` and `license.lic` — keep owner-only permissions and encrypt before copying offsite.

---

# STOP

```text
✓ backup.sh / backup.bat run manually once and produce a valid archive
✓ scheduled job confirmed to fire
✓ old backups get pruned instead of filling the disk
✓ a restored backup (config/ + data/ + media/ dropped back in) brings the service back to a working state
✓ frontend/ was correctly excluded — a restore doesn't need it, a fresh `npm run build` does
```

---

<a id="phase-8"></a>
# Phase 8 — Frontend Hosted on the Server + Electron Thin Client

This is where the architecture actually diverges from v5. Three things happen in this phase, in order: the frontend gets built for **server** hosting instead of per-client bundling; it gets wrapped in its own OS service and unified with the backend behind Nginx; and Electron is rewritten from "container that ships the UI" to "thin, licensed window onto the UI."

## Task 8.1 — Why Runtime Config for the Frontend Itself Is No Longer Needed

v5's Task 8.1 explained why `NEXT_PUBLIC_*` doesn't work: it's inlined at build time, and every client PC might need a different backend address, so one build can't serve every machine.

That constraint is gone here, because the frontend is no longer built once per client — **it's built exactly once, for exactly one target: the server it's about to run on.** It doesn't need to know a LAN IP at all, at build time or run time, because it talks to the backend through a relative path (`/api/...`) that resolves against whatever address the browser (or Electron) used to load the page in the first place — see Task 8.4. There is nothing analogous to `config.env` needed on the frontend side.

The runtime-config problem doesn't vanish, though — it moves to Electron, which still doesn't know the server's LAN address until install time (Task 8.5).

## Task 8.2 — Build the Frontend for Server Hosting

Use Next.js's `standalone` output instead of `export` — the server can run a real Node process, so there's no reason to give up SSR/server actions/API routes the way v5 did to avoid needing Node on every client PC.

```js
// next.config.js
module.exports = {
  output: 'standalone',
};
```

```bash
npm run build
```

This produces `.next/standalone/` (a minimal, self-contained Node server), `.next/static/`, and `public/`. Assemble them into the server layout from Task 1.1:

```bash
mkdir -p app-server/frontend
cp -r .next/standalone/. app-server/frontend/
mkdir -p app-server/frontend/.next
cp -r .next/static app-server/frontend/.next/static
cp -r public app-server/frontend/public
```

This is also the first point in the plan where a frontend config file actually gets written. The frontend needs almost nothing, because it doesn't call the backend by IP — it calls it by relative path (`/api/...`), resolved against whatever origin Nginx served the page from (Task 8.4). All it needs is which local port to listen on:

```env
# config/frontend.env
NODE_ENV=production
HOST=127.0.0.1
PORT=3000
```

```bash
chmod 600 app-server/config/frontend.env
chown app-svc:app-svc app-server/config/frontend.env
```

`HOST=127.0.0.1` matters: the frontend process should only be reachable from Nginx on the same machine, never directly from the LAN — see Phase 11.

Smoke-test it directly before wrapping it in anything:

```bash
cd app-server/frontend
PORT=3000 HOSTNAME=127.0.0.1 node server.js
```

> If your app genuinely has no server-side needs (pure static UI, no SSR, no server actions), `output: 'export'` served as static files directly by Nginx is a lighter alternative to running a Node process at all — skip Task 8.3's service entirely and add a `location /` block in Task 8.4 pointing at the exported files instead of proxying to a Node port. The rest of this phase assumes the standalone-server path, since it keeps the door open for SSR later.

## Task 8.3 — Run the Frontend as Its Own Service

Same pattern as Phase 6, applied to the frontend, placed here because this is the first point in the plan where `app-server/frontend/server.js` exists to point at.

### Linux (systemd)

```ini
# /etc/systemd/system/app-frontend.service
[Unit]
Description=App Frontend (Next.js)
After=network.target app.service

[Service]
Type=simple
User=app-svc
Group=app-svc
WorkingDirectory=/opt/app/app-server/frontend
Environment=PORT=3000
Environment=HOSTNAME=127.0.0.1
Environment=NODE_ENV=production
ExecStart=/usr/bin/node /opt/app/app-server/frontend/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable app-frontend.service
sudo systemctl start app-frontend.service
sudo systemctl status app-frontend.service
```

### Windows (NSSM)

```powershell
nssm install AppFrontend "C:\Program Files\nodejs\node.exe"
nssm set AppFrontend AppParameters "C:\AppServer\frontend\server.js"
nssm set AppFrontend AppDirectory "C:\AppServer\frontend"
nssm set AppFrontend AppEnvironmentExtra PORT=3000 HOSTNAME=127.0.0.1 NODE_ENV=production
nssm set AppFrontend Start SERVICE_AUTO_START
nssm set AppFrontend AppStdout C:\AppServer\logs\frontend-stdout.log
nssm set AppFrontend AppStderr C:\AppServer\logs\frontend-stderr.log
nssm set AppFrontend ObjectName .\app-svc <service-account-password>
nssm start AppFrontend
```

Same low-privilege service account as the backend (`app-svc`), not `LocalSystem`/admin.

`After=app.service` (Linux) is a soft ordering hint, not a hard dependency — the frontend process starts fine even if the backend isn't up yet; individual page requests will just get API errors until the backend catches up. It's there so a normal boot sequence brings the backend up first in the common case.

## Task 8.4 — Unify Frontend + API Behind One Origin

This is the piece deferred from Phase 2's Task 2.3b, added now that both services actually exist. Extend the same Nginx config:

```nginx
# /etc/nginx/sites-available/app-server — extended from Phase 2
server {
    listen 80;
    server_name app-server.local;

    location /static/ {
        alias /opt/app/app-server/staticfiles/;
    }

    location /media/ {
        alias /opt/app/app-server/media/;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # NEW — everything else goes to the Next.js frontend
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

With this in place, `http://192.168.1.100/` serves the app UI and `http://192.168.1.100/api/...` serves the API, both from the same origin. That's what eliminates the CORS requirement for browser-side calls (Task 2.4) and gives Electron exactly one address to remember (Task 8.5), instead of one for the UI and one for the API.

## Task 8.5 — Electron Becomes a Thin Client

The Electron main process no longer loads a bundled Next.js export. It still does the exact same pre-flight license check as v5, then points a native window at the server instead of a local file.

### Runtime config: `config.env` in `userData`

Unchanged mechanism from v5 — `userData` is user-writable without admin rights, survives app updates, and lives outside the packaged app. Only the contents change:

```env
# Single LAN address for the whole app — frontend UI and API both live behind this one origin
SERVER_URL=http://192.168.1.100

# Optional feature flags the Electron shell itself reads (most flags now live in the frontend instead)
ENABLE_TELEMETRY=false
```

### Electron main process

```js
// electron/main.js
const { app, BrowserWindow, ipcMain } = require('electron');
const path = require('path');
const fs = require('fs');
const dotenv = require('dotenv');
const { URL } = require('url');

let mainWindow;
let runtimeConfig = {};

function loadOrCreateConfig() {
  const userDataPath = app.getPath('userData');
  const configPath = path.join(userDataPath, 'config.env');

  if (!fs.existsSync(configPath)) {
    const templatePath = path.join(process.resourcesPath, 'default.env');
    if (fs.existsSync(templatePath)) {
      fs.copyFileSync(templatePath, configPath);
    } else {
      fs.writeFileSync(configPath, 'SERVER_URL=http://localhost\n');
    }
  }

  const parsed = dotenv.config({ path: configPath }).parsed || {};
  runtimeConfig = parsed;
  return runtimeConfig;
}

ipcMain.handle('get-config', () => runtimeConfig);

ipcMain.handle('set-config', (event, newValues) => {
  const userDataPath = app.getPath('userData');
  const configPath = path.join(userDataPath, 'config.env');
  const merged = { ...runtimeConfig, ...newValues };
  const serialized = Object.entries(merged).map(([k, v]) => `${k}=${v}`).join('\n');
  fs.writeFileSync(configPath, serialized + '\n');
  runtimeConfig = merged;
  return merged;
});

async function checkServerLicense(serverUrl) {
  try {
    const res = await fetch(`${serverUrl}/api/license/status/`, { signal: AbortSignal.timeout(5000) });
    if (!res.ok) return { valid: false, reason: 'http_error' };
    return await res.json();
  } catch (e) {
    return { valid: false, reason: 'unreachable' };
  }
}

function showFallback(reason) {
  const errWindow = new BrowserWindow({
    width: 480,
    height: 320,
    resizable: false,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });
  errWindow.loadFile(path.join(__dirname, 'license-invalid.html'));
  errWindow.webContents.on('did-finish-load', () => {
    errWindow.webContents.send('license-error', { valid: false, reason });
  });
  return errWindow;
}

function openMainWindow(serverUrl) {
  mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      webSecurity: true,
    },
  });

  // NEW — lock navigation to the configured server origin. Content is remote now,
  // not a bundled local file, so nothing prevents the loaded page from trying to
  // carry the shell elsewhere unless we explicitly stop it.
  const allowedOrigin = new URL(serverUrl).origin;
  mainWindow.webContents.on('will-navigate', (event, targetUrl) => {
    if (new URL(targetUrl).origin !== allowedOrigin) {
      event.preventDefault();
    }
  });
  mainWindow.webContents.setWindowOpenHandler(() => ({ action: 'deny' }));

  // NEW — the license check can pass and the page can still fail to load a moment
  // later (frontend service crashed, Nginx down, LAN blip). That's a different
  // failure than "license invalid" and needs its own fallback.
  mainWindow.webContents.on('did-fail-load', (event, errorCode, errorDescription) => {
    if (errorCode === -3) return; // ERR_ABORTED from a normal navigation cancel — ignore
    mainWindow.close();
    showFallback('unreachable');
  });

  mainWindow.loadURL(serverUrl);
}

ipcMain.handle('retry-license-check', async () => {
  const cfg = loadOrCreateConfig();
  const serverUrl = cfg.SERVER_URL || 'http://localhost';
  const status = await checkServerLicense(serverUrl);
  if (status.valid) {
    BrowserWindow.getAllWindows().forEach((w) => w.close());
    openMainWindow(serverUrl);
  }
  return status;
});

async function startApp() {
  const cfg = loadOrCreateConfig();
  const serverUrl = cfg.SERVER_URL || 'http://localhost';
  const status = await checkServerLicense(serverUrl);

  if (!status.valid) {
    showFallback(status.reason);
    return;
  }

  openMainWindow(serverUrl);
}

app.whenReady().then(startApp);
```

The pre-flight license check is unchanged in spirit from v5 — it still runs before any application window opens, it still calls `/api/license/status/`, and it still shows a local fallback screen on failure. Everything below it is new: the navigation guard, the `did-fail-load` handler, and `loadURL` in place of `loadFile`.

### Preload bridge

```js
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('appConfig', {
  get: () => ipcRenderer.invoke('get-config'),
  set: (newValues) => ipcRenderer.invoke('set-config', newValues),
  onLicenseError: (callback) => ipcRenderer.on('license-error', (_event, data) => callback(data)),
  retryLicenseCheck: () => ipcRenderer.invoke('retry-license-check'),
});
```

Unchanged from v5 — kept for the fallback window and for the Settings screen (Task 8.6). The main application window itself, being remote content now rather than bundled renderer code, generally won't use this bridge for its own logic; it talks to the backend directly via relative `fetch('/api/...')` calls, same-origin, no IPC required.

### `electron/license-invalid.html`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>License Problem</title>
  <style>
    body { font-family: system-ui, sans-serif; padding: 24px; background: #1e1e1e; color: #f5f5f5; }
    h1 { font-size: 18px; margin-bottom: 8px; }
    #reason { color: #ff8080; margin: 12px 0 20px; line-height: 1.4; }
    button { padding: 8px 16px; cursor: pointer; }
  </style>
</head>
<body>
  <h1>Can't connect to the app server</h1>
  <p id="reason">Checking server status...</p>
  <button id="retry">Retry</button>

  <script>
    const reasons = {
      missing: "No license file found on the server.",
      expired: "The server's license has expired.",
      hwid_mismatch: "The license doesn't match this server.",
      tampered: "The license file failed verification.",
      unreachable: "Can't reach the server. Check the server address in Settings.",
      http_error: "The server returned an error.",
    };

    function showReason(status) {
      document.getElementById('reason').textContent =
        reasons[status.reason] || `Problem: ${status.reason}`;
    }

    window.appConfig.onLicenseError(showReason);

    document.getElementById('retry').addEventListener('click', async () => {
      const status = await window.appConfig.retryLicenseCheck();
      if (!status.valid) {
        showReason(status);
      }
    });
  </script>
</body>
</html>
```

Same file as v5, same reasons dictionary — `unreachable` now covers two distinct causes underneath (license check couldn't reach the server at all, or the app page failed to load after a successful check), which is fine, since from the user's point of view both mean "can't connect right now, check the network/server and retry."

## Task 8.6 — In-App Settings Screen

```ts
// Settings page, running inside the loaded frontend, calling back into Electron via IPC
await window.appConfig.set({ SERVER_URL: 'http://10.0.0.5' });
// Then prompt: "Restart now to apply changes?"
```

This screen is now served by the frontend (part of the normal app UI), not bundled Electron renderer code — but it still needs the `appConfig` bridge, since only the Electron main process can write `config.env` in `userData`. Expose the same `contextBridge` API to the remotely-loaded page (this is safe precisely because navigation is locked to `SERVER_URL`'s origin via the guard in Task 8.5 — nothing else the shell could load can reach this bridge).

## Task 8.7 — Package With `electron-builder`

```bash
npm install --save-dev electron-builder dotenv
```

```json
{
  "main": "electron/main.js",
  "build": {
    "appId": "com.yourcompany.appclient",
    "productName": "App Client",
    "files": ["electron/**/*"],
    "extraResources": [{ "from": "default.env", "to": "default.env" }],
    "win": { "target": "nsis", "icon": "build/icon.ico" },
    "nsis": {
      "oneClick": true,
      "perMachine": false,
      "allowToChangeInstallationDirectory": false
    }
  }
}
```

```bash
npx electron-builder --win
```

The only structural difference from v5's build config: no `"out/**/*"` in `files` — there is no Next.js renderer bundle to ship, so the resulting installer is a fraction of the size.

## Task 8.8 — Test the Packaged App

```text
✓ App installs without admin prompt
✓ App starts, checks the license against SERVER_URL, and loads the app UI live from the server
✓ Editing config.env + restarting picks up the new SERVER_URL
✓ Logs land in %APPDATA%\AppClient\logs\
✓ License-invalid screen shows the right message for each failure reason
✓ Retry button re-checks and opens the main window once the license is valid again
✓ Killing the frontend service (app-frontend.service) while the shell is open triggers the
  did-fail-load fallback, not a blank/crashed window
✓ Attempting to navigate the loaded page to a different origin is blocked
✓ window.open()-style popups from the loaded page are blocked, not opened as new native windows
```

---

# STOP

```text
✓ Frontend builds once, runs as its own service on the server, and is reachable through Nginx
✓ Nginx unifies frontend + API behind one origin — no CORS preflight for browser-side calls
✓ Electron app packaged as one-click installer, with no bundled application UI inside it
✓ Runtime config.env editable post-build, now holding SERVER_URL instead of API_URL
✓ App reaches the real, compiled, serviced backend AND the real, serviced frontend on the LAN
✓ license-invalid.html exists and renders correctly, with a working retry path
✓ Navigation guard and did-fail-load fallback both verified working
```

---

<a id="phase-9"></a>
# Phase 9 — Frontend Licensing Gate in Electron

The gating logic is the same shape as v5: the Electron main process calls `/api/license/status/` **before** opening any application window (already wired in Phase 8's `startApp()`). If invalid, it shows the local fallback with the reason and a working retry button instead of the app. What's different is where the app itself lives once the gate passes.

The remotely-loaded page doesn't need to know about licensing any more than v5's local renderer did — if the main window opened, the license was valid at that moment. Backend API calls still return 401/403 if the license is revoked mid-session, handled the same way: a generic "session expired" dialog in the page.

**One genuine improvement from centralizing the frontend:** there is now exactly one deployment of the app UI to keep in sync with license state, instead of one per client PC. Fixing a licensing-adjacent UI bug (say, a wrong message on the renew screen) means redeploying `app-server/frontend/` once — every Electron shell picks it up automatically on its next load, with no redistribution step.

## Task 9.1 — Full-Stack Lifecycle Test

```text
✓ valid license          → main window opens, app runs normally
✓ missing license file    → fallback window with reason="missing"
✓ expired license         → fallback window with reason="expired"
✓ license for wrong hwid  → fallback window with reason="hwid_mismatch"
✓ tampered license file   → fallback window with reason="tampered"
✓ backend unreachable      → fallback window with reason="unreachable"
✓ frontend service down (backend fine) → fallback window via did-fail-load, reason="unreachable"
✓ (online mode) revoked   → blocked after grace period, all connected clients reflect it
✓ license revoked mid-session → next API call returns 403, page shows session-expired dialog
✓ fixing the license server-side, then clicking Retry → app opens without a restart
```

## Task 9.2 — Multi-Client Test

Install the Electron shell on **3 different client PCs** on the same LAN, all pointing at the same `SERVER_URL`:

```text
✓ All three can log in and use the app concurrently, all served from the same frontend build
✓ If max_clients is set in the license and exceeded, the 4th client gets a clear "client limit reached" message
✓ Restarting the frontend service on the server briefly disconnects all three clients the same way,
  and all three recover independently via their own retry paths — no client-specific state to fix
```

---

# STOP

```text
✓ Electron client reflects license status correctly for every backend AND frontend state
✓ Multiple client PCs work concurrently against the same server-hosted frontend
```

---

<a id="phase-10"></a>
# Phase 10 — Two Installers

## Server installer (`app-server-setup.exe`)

Bundles everything from v5's server installer, **plus** the frontend build and its service registration:
- Compiled backend binary (`backend.exe` / `backend.bin`)
- Next.js standalone frontend build (`frontend/`)
- `default.env` / `frontend.env` templates
- `public_key.pem`
- `scripts/get_machine_id.py`, `scripts/backup.sh` or `.bat`
- NSSM service registration (Windows) or systemd unit (Linux) for **both** `app.service` and `app-frontend.service`
- Nginx site config wiring both services behind one origin (or documented manual steps, if Nginx isn't bundled by the installer)
- Firewall rule for the Nginx-facing port only (see Phase 11)

On first run, the installer:
1. Asks for install directory (default `C:\AppServer\` or `/opt/app/app-server`).
2. Runs `get_machine_id.py` automatically and displays the HWID with a "Copy" button.
3. Asks the operator to drop `license.lic` into `config\`.
4. Registers and starts **both** services, and reloads Nginx.
5. Verifies `/api/license/status/` returns `valid: true`, and that `/` (the frontend) loads through the same origin.

## Client installer (`app-client-setup.exe`)

Produced by `electron-builder` in Phase 8. This is where the biggest practical change shows up: it bundles **only the Electron shell** — no Next.js build, no renderer bundle, just:
- The Electron main process, preload script, and fallback HTML.
- `default.env` template (copied to `%APPDATA%\AppClient\config.env` on first launch).

On first run, the app:
1. Asks the user for the **server's LAN address** (or auto-discovers via mDNS, optional).
2. Writes `SERVER_URL=http://<server-ip>` to `config.env`.
3. Restarts and connects — at which point it's loading the exact same live UI every other client on the LAN is loading.

Still one-click from the user's perspective: install → enter server address → use. The installer itself, however, is dramatically smaller than v5's, since it isn't shipping an application UI at all.

## Distribution

- Server installer: delivered once to IT (or installed by you on-site) — now sets up two services instead of one, plus Nginx.
- Client installer: distributed the same way as v5 (shared network drive, USB stick, internal download link), just a much smaller file. Frontend updates no longer require touching this installer at all — only backend or Electron-shell changes do.

---

# STOP

```text
✓ Server installer: one machine, registers both services, license validates, Nginx unifies the origin
✓ Client installer: any number of machines, each with its own config.env holding SERVER_URL
✓ A frontend-only bug fix ships by redeploying app-server/frontend/ — zero client-side redistribution
✓ Zero-touch from server side for additional client installs
```

---

<a id="phase-11"></a>
# Phase 11 — Security Hardening (Audit Pass)

Same verification-pass philosophy as v5 — most of this was implemented earlier under "do this now, not at the end." A few items are genuinely new because content is now loaded from the network rather than bundled locally; the rest carry over unchanged.

### Backend (server) — unchanged from v5
- `DEBUG=False` — confirm in the live `.env`.
- Secure, unique `SECRET_KEY` per deployment.
- `data/db.sqlite3` permissions restricted to the service account.
- Confirm `config/.env`, `config/frontend.env`, and `config/license.lic` are still `chmod 600`, owned by the service account — re-check after every license re-issue, backup restore, or fresh install.
- Online mode: phone-home calls must be HTTPS, license-server endpoint rate-limited against HWID-guessing.

### Backend + Frontend network exposure — updated for this variant
- **Close ports 8000 and 3000 to the LAN.** Both the backend (`waitress`) and the frontend (`next start` / standalone `server.js`) should bind to `127.0.0.1` only, reachable exclusively through Nginx. Undo Task 2.6's firewall rule that opened 8000 directly — that was only ever meant for Phase 2's standalone testing, before Nginx existed.
- **Open only the Nginx-facing port** (80, or 443 if TLS is configured) on the firewall:
  ```bash
  sudo ufw delete allow 8000/tcp
  sudo ufw allow 80/tcp
  ```
  ```powershell
  netsh advfirewall firewall delete rule name="App Server API"
  netsh advfirewall firewall add rule name="App Server HTTP" dir=in action=allow protocol=TCP localport=80
  ```
- **HTTPS matters more here than it did in v5.** Previously only API traffic crossed the network in the clear; now the entire application UI does too, on every page load. If the LAN spans untrusted segments (guest Wi-Fi, shared switches, anything outside a controlled closet), terminate TLS at Nginx, same certificate/config mechanism v5 already mentioned for the API alone.
- CORS: `CORS_ALLOWED_ORIGINS` locked to the Nginx origin, never `"*"` — same rule as v5, now doubling as a backstop for direct calls to the (closed) backend port rather than the primary defense.

### Frontend (client PCs) — updated for this variant
- `contextIsolation: true`, `nodeIntegration: false` — confirmed on **both** the main window and the fallback window, same as v5.
- `webSecurity: true` (default — do not disable), same as v5.
- **CSP moves server-side.** In v5, Electron controlled the renderer's Content-Security-Policy because it shipped the HTML. Here, Electron is loading someone else's (your own server's) page over the network — it doesn't own the markup. Set the CSP as a real HTTP response header from Nginx or Next.js instead:
  ```nginx
  add_header Content-Security-Policy "default-src 'self'; connect-src 'self'; frame-ancestors 'none';" always;
  ```
- **New: navigation is locked to the server's origin.** Implemented in Task 8.5 via `will-navigate` and `setWindowOpenHandler`. This didn't exist in v5 because bundled local content had nowhere untrusted to navigate to; remote content does, so this guard is mandatory here, not optional.
- **New: the frontend Node process binds to `127.0.0.1`**, never `0.0.0.0` — it should be unreachable except through Nginx on the same machine, exactly like the backend.
- Don't log `config.env` contents to disk. `config.env` now carries less (`SERVER_URL` plus a couple of flags, no API keys duplicated), so the blast radius of a leaked copy is smaller than it was in v5, but the rule stands.
- If using `electron-updater`: verify update signatures (`publisherName` in electron-builder config) — this now only needs to cover the small Electron shell, not an application bundle.
- `config.env` in `%APPDATA%` is readable by that user only by default — don't weaken it.

---

<a id="milestone-order"></a>
# Milestone Order

1. **Phase 1–2** — backend runs correctly, uncompiled, reachable from another machine, CORS/ALLOWED_HOSTS driven by `.env`, Nginx serving static/media/API (no frontend block yet).
2. **Phase 3** — logging wired up for the backend.
3. **Phase 4** — backend licensing enforced and fully tested, still uncompiled, HWID hand-off defined.
4. **Phase 5** — compile the backend with Nuitka, re-test licensing against the compiled binary.
5. **Phase 6** — wrap the *compiled backend* in a systemd/NSSM service.
6. **Phase 7** — backups wired around the live, serviced backend.
7. **Phase 8** — frontend built for server hosting, wrapped in its own service, unified with the backend behind Nginx, and the Electron shell rewritten as a thin client pointing at that unified origin, with its license-invalid screen, retry path, navigation guard, and did-fail-load fallback all functional. **Server track and frontend build both finish here.**
8. **Phase 9** — frontend license gate re-verified end-to-end, multi-client test across several PCs against the one shared frontend.
9. **Phase 10** — two separate installers produced and tested end-to-end (server installer now sets up two services; client installer ships no application UI at all).
10. **Phase 11** — hardening audit across backend, frontend, and Electron shell, including closing the raw 8000/3000 ports and moving CSP server-side.

Each phase's STOP checkpoint should pass before starting the next one. Backend and frontend remain two independently deployable services on the server, with their own config files and their own service units, unified only at the Nginx layer — the Electron shell's config is editable post-build via `config.env` in `userData` + IPC, and now holds a single `SERVER_URL` rather than a separate API address.

---

<a id="future-feature"></a>
# Future Feature — License Renewal Endpoint

Unchanged from v5 in mechanism — applies to any project built on this template. The one adjustment worth calling out is where the renewal *UI* now lives.

## The problem

Same as v5: renewing an expired or changed license today means generate a new `license.lic` → send it to the customer/IT → someone manually overwrites `config/license.lic` on the server → restart the service.

## Two ways to solve it

### A. Push model — an authenticated renewal endpoint

Add `POST /api/license/renew/` that accepts a new signed token and applies it live, without a manual file swap or service restart.

**Design constraints (unchanged from v5):**
- Exempt from the license-required middleware.
- Doesn't rely on normal user auth — use a separate static admin token (`LICENSE_ADMIN_TOKEN`) or a dedicated Django superuser check.
- Validates the incoming token exactly like startup does (signature, HWID) before writing anything.
- Writes atomically (temp file + `os.replace`).
- Logs the renewal to `license.log`.

```python
# views.py
import os, tempfile, logging
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAdminUser
from rest_framework.response import Response

from .license_check import validate_license_token, current_hwid, get_public_key, validate_license

logger = logging.getLogger("license")

@api_view(["POST"])
@permission_classes([IsAdminUser])
def renew_license(request):
    new_token = request.data.get("license_token", "").strip()
    if not new_token:
        return Response({"valid": False, "reason": "missing_token"}, status=400)

    public_key = get_public_key(CONFIG_DIR)
    try:
        payload = validate_license_token(new_token, public_key)
    except Exception:
        return Response({"valid": False, "reason": "tampered"}, status=400)

    if payload["hwid"] != current_hwid():
        return Response({"valid": False, "reason": "hwid_mismatch"}, status=400)

    try:
        current = validate_license(CONFIG_DIR)
        if current["license_id"] != payload["license_id"] and not request.data.get("force"):
            return Response({"valid": False, "reason": "license_id_mismatch"}, status=400)
    except Exception:
        pass

    license_path = os.path.join(CONFIG_DIR, "license.lic")
    fd, tmp_path = tempfile.mkstemp(dir=CONFIG_DIR)
    with os.fdopen(fd, "w") as f:
        f.write(new_token)
    os.replace(tmp_path, license_path)
    os.chmod(license_path, 0o600)

    logger.info(f"License renewed via API: {payload['license_id']} exp={payload['exp']}")
    return Response({
        "valid": True,
        "client": payload["client"],
        "exp": payload["exp"],
        "features": payload["features"],
    })
```

```python
# urls.py — exempt from the license-required middleware alongside /api/license/status/
urlpatterns = [
    path("api/license/status/", license_status_view),
    path("api/license/renew/", renew_license),
]
```

### B. Pull model — automatic renewal for online-mode deployments

Unchanged from v5:

- A scheduled job calls `payload["server_url"]` with `license_id` + `hwid` on a daily interval.
- A newer signed token runs through the same validation and atomic-write path as the push endpoint, factored into a shared `apply_new_license(token)` function.
- Unreachable license server falls back to `grace_days` handling.

```python
# license_renewal.py
def apply_new_license(token, config_dir=CONFIG_DIR):
    """Shared by both the push endpoint and the pull/cron job."""
    public_key = get_public_key(config_dir)
    payload = validate_license_token(token, public_key)
    if payload["hwid"] != current_hwid():
        raise ValueError("hwid_mismatch")

    license_path = os.path.join(config_dir, "license.lic")
    fd, tmp_path = tempfile.mkstemp(dir=config_dir)
    with os.fdopen(fd, "w") as f:
        f.write(token)
    os.replace(tmp_path, license_path)
    os.chmod(license_path, 0o600)
    logger.info(f"License renewed: {payload['license_id']} exp={payload['exp']}")
    return payload
```

## Frontend hook-up — the part that changes here

In v5, the "Renew License" screen was part of the renderer bundle shipped inside every Electron install, and it called Electron's `retryLicenseCheck()` IPC handler to reflect the renewed state without a restart.

Here, that screen is just a normal page in the one, centrally-hosted frontend (Task 8.2) — implemented once, not once per client. It calls the renewal endpoint the same way any other page calls the API (relative `fetch('/api/license/renew/')`, same-origin through Nginx):

```ts
async function renewLicense(token: string) {
  const res = await fetch('/api/license/renew/', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${adminToken}` },
    body: JSON.stringify({ license_token: token }),
  });
  return await res.json();
}
```

The Electron shell's `retryLicenseCheck()` IPC handler still exists (Task 8.5) and is still useful — if a fallback window happens to be showing on a given client PC when the license gets renewed, that client's Retry button will pick up the change on the next click, same as v5. But the renew action itself no longer needs to know it's running inside Electron at all; it's ordinary web app code, and it now fixes the state for every connected client the moment it runs, not just the one that clicked it.

## Test checklist

```text
✓ /api/license/renew/ works even while the currently-installed license is expired
✓ endpoint rejects a token signed with the wrong key
✓ endpoint rejects a token whose hwid doesn't match this machine
✓ endpoint rejects a token for a different license_id unless force=true is passed
✓ a crash mid-write never leaves license.lic empty or corrupted (atomic replace holds)
✓ renewal is logged to license.log with the new license_id and exp
✓ (pull model) scheduled job picks up a newer token automatically within one interval
✓ (pull model) license server unreachable → falls back to grace_days, no crash
✓ the shared frontend's "Renew License" page updates state without any client-specific code
✓ an Electron shell showing the fallback window at the time of renewal recovers via its existing Retry button
```
