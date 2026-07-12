# Full-Stack App — Deployment Plan (Server-Hosted Frontend Variant)

Django backend (server) + Next.js frontend (**also server**) + Tauri thin client (client PCs), LAN-deployed, Nuitka-compiled backend, licensed.

Backend and frontend are now **two services on the same server machine**, plus a separate, much lighter Tauri shell shipped to every workstation. Each client PC no longer carries its own copy of the frontend build — it opens a small native window that loads the frontend live from the server over the LAN, the same way a browser would, but wrapped in a controlled, licensed, native shell instead of a bare browser tab.

> This deployment plan is for clients whose local server is powerful enough to run the frontend as well as the backend, instead of shipping a full Next.js build to every PC. Everywhere you see `app-server`, `app-client`, `AppClient`, `AppServer`, or `com.yourcompany.appclient`, swap in your actual project's name.

---

## Index

1. [Phase 1 — Two Project Structures (Server vs. Client)](#phase-1)
2. [Phase 2 — Make the Backend Production Ready](#phase-2)
3. [Phase 3 — Logging](#phase-3)
4. [Phase 4 — Backend Licensing (py-rizmi)](#phase-4)
5. [Phase 5 — Compile Backend With Nuitka](#phase-5)
6. [Phase 6 — Windows Service / Linux Service (Backend)](#phase-6)
7. [Phase 7 — Backup Strategy](#phase-7)
8. [Phase 8 — Frontend Hosted on the Server + Tauri Thin Client](#phase-8)
9. [Phase 9 — Frontend Licensing Gate in Tauri](#phase-9)
10. [Phase 10 — Two Installers (Server + Client)](#phase-10)
11. [Phase 11 — Security Hardening (Audit Pass)](#phase-11)
12. [Milestone Order](#milestone-order)
13. [Future Feature — License Renewal Endpoint](#future-feature)

---

<a id="phase-1"></a>
# Phase 1 — Two Project Structures (Server vs. Client)

Goal: separate the deployment artifacts for the server (backend **and now frontend**) from the client PCs (Tauri shell only) from day one. They will never ship together.

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

## Task 1.2 — Frontend (Client) Deployment Layout — Now Just a Tauri Shell

```text
app-client/
├── app/                          # Tauri shell only — no bundled application UI
│   ├── AppClient.exe             # the Tauri-generated native binary (webview launcher)
│   ├── resources/
│   │   └── default.env           # template copied to the app config dir on first run
│   └── ... (WebView2 runtime on Windows / WebKitGTK on Linux / WKWebView on macOS)
└── (per-user runtime data — NOT shipped)
    └── %APPDATA%/com.yourcompany.appclient/
        ├── config.env            # <-- USER-EDITABLE, now holds SERVER_URL
        └── logs\
```

There is no packaged Next.js build inside the Tauri app at all. Unlike Electron, Tauri doesn't bundle its own copy of Chromium either — it drives whatever web-rendering engine already ships with the OS (WebView2, WebKitGTK, or WKWebView), which is part of why the installer stays small. The shell's only job is to (1) gate on the license, (2) point a native window at the server, and (3) show a local fallback screen if it can't. The actual application — every screen, every component — is fetched live from `SERVER_URL` each time the shell opens. Fixing a frontend bug means redeploying `app-server/frontend/` once; it does not mean rebuilding or redistributing anything to client PCs.

The user-editable file is **`config.env`**, stored in the app's config directory (resolved via Tauri's `path` API, keyed off the `identifier` in `tauri.conf.json` — e.g. `com.yourcompany.appclient`) on each client PC — see Phase 8 for the mechanism.

## Task 1.3 — Move SQLite Outside Backend Code

```text
backend/db.sqlite3  ->  app-server/data/db.sqlite3
```

## Task 1.4 — Move Media Outside Backend Code

```text
backend/media/  ->  app-server/media/
```

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

`config/license.lic` and `config/public_key.pem` are fixed, hardcoded filenames under `CONFIG_DIR` — no env var needed for them.

**Path resolution:** resolve every backend path from one explicit root, not `__file__`, since Nuitka `--onefile` extracts to a temp folder at runtime.

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
✓ client config directory identified as the home of the runtime config.env
✓ app-client's planned layout has no bundled application UI — confirmed shell-only
✓ nothing under app-server/frontend/ has been created or configured yet — that's Phase 8, after Phase 7
```

---

<a id="phase-2"></a>
# Phase 2 — Make the Backend Production Ready

One item is deferred and called out explicitly below (Task 2.3b).

## Task 2.1 — Install Dependencies

```bash
pip install waitress python-dotenv whitenoise django-cors-headers
```

`django-cors-headers` stays installed even with the unified-origin approach in Phase 8 — keep it as defense-in-depth (Task 1.5), and it's the only thing standing between the Tauri client and the API if you choose *not* to unify origins (see Task 2.3b).

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

**Option B — Authenticated Django view** streaming files from `MEDIA_DIR`.

### Task 2.3b — The Frontend `location /` Block Comes Later, On Purpose

This same config file gets one more `location` block in **Phase 8, Task 8.4**, once the Next.js frontend actually exists to proxy to. Adding it now would mean reloading Nginx with a `proxy_pass` pointing at a port nothing is listening on yet — the same class of mistake as wiring a service to a binary that hasn't been compiled yet. This phase intentionally stops at static/media/API only.

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

**How this plays out once Phase 8 is done:** if you go with the unified Nginx origin (Task 8.4, recommended), every request the browser makes to `/api/...` is same-origin as far as the browser is concerned — it never triggers a CORS preflight at all, because it's not cross-origin. `CORS_ALLOWED_ORIGINS` still matters as a second line of defense, since Django's own port (8000) is still technically reachable from the LAN unless you also close it off in Phase 11. If you skip the unified origin and keep the frontend on its own port with no reverse proxy in front of it, CORS is back to being load-bearing — lock `CORS_ALLOWED_ORIGINS` to that port's origin.

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

`host="0.0.0.0"` is fine here because Phase 11 will bind this to `127.0.0.1` once Nginx is fronting it — see Task 2.6 and Phase 11.

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

This phase governs **backend (Django) logging only**. The Next.js frontend's process logging is handled separately by its own service wrapper in Phase 8, writing to `logs/frontend-stdout.log` / `logs/frontend-stderr.log` alongside the files below.

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
# Phase 4 — Backend Licensing (py-rizmi)

The licensing model still binds the license to the **server machine** only — and that server machine now happens to run the frontend too, so one license continues to cover the whole deployment, not per-client licensing.

This phase uses **[py-rizmi](https://github.com/Ramzi-Hadrouk/py-rizmi)**, an offline RSA-signed JWT licensing toolkit, instead of hand-rolling the keypair/issuance/validation scripts. It covers the same license fields as before (`client`, `license_id`, `hwid`, `features`, `max_clients`, `mode`, `server_url`, `grace_days`, `iat`, `exp`) via a PyQt6 GUI, headless CLI scripts, and a pure-Python core module — so key generation and license issuance become point-and-click (or single-command) instead of scripts you have to maintain yourself.

## Task 4.1 — Design the License Token

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

`max_clients` is informational — actual concurrent-client enforcement (Tauri windows hitting the same backend) is a backend concern. Every one of these fields is a bound input widget in py-rizmi's License Generation view, so there's no hard-coded payload data to keep in sync by hand.

## Task 4.2 — Get py-rizmi and Install Its Dependencies

```bash
git clone https://github.com/Ramzi-Hadrouk/py-rizmi.git
cd py-rizmi
pip install -r requirements.txt
```

The `src/core/` layer (`hwid.py`, `keypair.py`, `license_token.py`, `license_issuer.py`, `license_validator.py`) is pure Python with zero GUI dependency — only `pyjwt` and `cryptography` are actually required at runtime. Vendor `src/core/` (and, if you use the drop-in helper, `backend/`) into the Django project; leave the PyQt6 GUI code out of anything that gets shipped or compiled with Nuitka in Phase 5.

## Task 4.3 — Generate a Signing Keypair (you keep this, once)

Either launch the GUI (`python main.py` → **Key Management** → pick a key size → **Generate**) or run the CLI headlessly:

```bash
python scripts/gen_keypair.py \
  --private-out keys/private_key.pem \
  --public-out keys/public_key.pem \
  --key-size 2048
```

- `private_key.pem` — stays on your authoring machine, never shipped.
- `public_key.pem` — ships with the backend, goes in `app-server/config/public_key.pem`.

## Task 4.4 — The HWID Hand-off Workflow

1. During server install, run `python scripts/get_machine_id.py` on the target machine (or use the GUI's **Machine ID** view and its **Copy HWID** button).
2. Whoever is on-site copies the printed HWID (SHA-256 hash) and sends it to you.
3. You paste that hash into `issue_license.py`'s `--hwid` flag (Task 4.5) — or the GUI's **License Generation** view.
4. You send back the resulting `license.lic` for them to place in `config/license.lic`.
5. If the server's hardware changes later, repeat this loop (or use the Future Feature renewal endpoint).

## Task 4.5 — Issue a License (your side, offline tool, not shipped)

```bash
python scripts/issue_license.py \
  --private-key keys/private_key.pem \
  --output license.lic \
  --client "Acme Corp" \
  --license-id "deploy-001" \
  --hwid "<paste-the-hwid-here>" \
  --features billing reports \
  --max-clients 10 \
  --grace-days 14 \
  --exp-days 365
```

Send the client `license.lic` → they drop it in `app-server/config/license.lic`, `chmod 600`, owned by the service account.

## Task 4.6 — Backend Validation Module

Vendor `src/core/license_validator.py` (and its `hwid.py` / `license_token.py` dependencies) into the Django project and use `LicenseValidator` directly:

```python
from src.core.license_validator import LicenseValidator

validator = LicenseValidator.from_file("app-server/config/public_key.pem")

try:
    payload = validator.validate_from_file("app-server/config/license.lic")
    print(f"Licensed to {payload.client}")
except ValueError as e:
    # one of: "missing", "expired", "tampered", "decode_error", "hwid_mismatch"
    print(f"License invalid: {e}")
```

`validate()` / `validate_from_file()` check signature, expiry, and HWID together, and log every outcome through the `license` logger already wired up in Phase 3 (`logs/license.log`). Only pass `check_hwid=False` for the read-only License Viewer use case — never in the actual startup/runtime gate.

py-rizmi also ships a server-side drop-in (`backend/license_check.py`) that wraps the same validator behind a plain function, if you'd rather not instantiate the class directly:

```python
from backend.license_check import validate_license

try:
    payload = validate_license("app-server/config")  # dir containing public_key.pem + license.lic
    print(payload["client"], payload["features"])
except ValueError as e:
    print(f"License invalid: {e}")
```

Wire whichever form you pick into Django as a startup check (Task 4.7) plus a lightweight cached middleware, and expose `/api/license/status/`:

```json
{ "valid": true, "client": "...", "exp": 1782864000, "features": ["billing","reports"] }
```

or

```json
{ "valid": false, "reason": "expired" }
```

This endpoint is what the Tauri client calls on boot (Phase 9), reached through the unified Nginx origin at `/api/license/status/`.

## Task 4.7 — Wire License Validation Into Startup

Fail fast: refuse to start serving traffic at all if the license doesn't validate, rather than letting the app come up and fail later on a request.

Add the check to `run_server.py` (Task 2.5), before `waitress.serve()` is ever called — this is the entry point Nuitka compiles in Phase 5 and systemd/NSSM launches in Phase 6, so it's the one place a fail-fast check covers every real deployment path:

```python
# run_server.py — add this before waitress.serve() is called
import sys
import logging
from src.core.license_validator import LicenseValidator

logger = logging.getLogger("license")

def enforce_license_or_exit(config_dir):
    try:
        validator = LicenseValidator.from_file(f"{config_dir}/public_key.pem")
        payload = validator.validate_from_file(f"{config_dir}/license.lic")
        logger.info(f"License OK: client={payload.client} exp={payload.exp}")
        return payload
    except ValueError as e:
        logger.error(f"Startup blocked: license check failed ({e})")
        print(f"FATAL: license check failed ({e}) — refusing to start.", file=sys.stderr)
        sys.exit(1)

enforce_license_or_exit(CONFIG_DIR)

# ...then the existing waitress.serve(application, host="0.0.0.0", port=port) call
```

Because this runs once at process start, systemd/NSSM (Phase 6) will see the process exit immediately with a non-zero code on an invalid license — `Restart=on-failure` will retry it (and keep failing, correctly, until the license is fixed) instead of the service silently hanging or serving a 500 on the first request.

If you also want the check to run under `manage.py runserver` during local development (not just the compiled `run_server.py` entry point), add the same call to `AppConfig.ready()` in the app's `apps.py`, guarded against Django's autoreloader running it twice:

```python
# apps.py
import os
from django.apps import AppConfig

class CoreConfig(AppConfig):
    name = "core"

    def ready(self):
        if os.environ.get("RUN_MAIN") != "false":
            from .license_check import enforce_license_or_exit
            enforce_license_or_exit(CONFIG_DIR)
```

The `run_server.py` hook is the one that matters for the compiled/serviced deployment target; the `apps.py` hook is a convenience for local development against the same license files.

## Task 4.8 — Optional Online Mode

`mode: "online"` triggers a periodic phone-home to `server_url`, using `grace_days` to tolerate flaky connections — both are ordinary payload fields in py-rizmi's License Generation view (or the CLI's `--mode` / `--server-url` flags), so no extra plumbing is needed to get them into the token itself; the phone-home job is still something you implement on the backend.

## Task 4.9 — Test Backend Licensing

```text
✓ valid license          → app runs normally
✓ missing license file    → clear error, logged to license.log
✓ expired license         → clear error, logged
✓ license for wrong hwid  → clear error, logged
✓ tampered license file   → rejected (signature check fails), logged
✓ malformed/corrupt token → decode_error, logged, doesn't crash the process
✓ (online mode) revoked license → blocked after grace period
✓ (online mode) offline for < grace period → still runs
✓ /api/license/status/ returns correct JSON for each state
✓ starting the service with an invalid license exits immediately (non-zero) instead of coming up
```

---

# STOP

```text
✓ backend refuses to run without a valid, matching license — verified via Task 4.7's exit-on-failure path
✓ /api/license/status/ exposes a clean JSON status the Tauri client can consume
✓ every failure path writes a line to logs/license.log
```

---

<a id="phase-5"></a>
# Phase 5 — Compile Backend With Nuitka

This phase compiles the **Python/Django backend only** — the Next.js frontend is a Node app, not a Python one, and doesn't go through Nuitka; it gets its own build step in Phase 8.

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

Notes:
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

This phase wraps the **compiled backend binary** from Phase 5 only. The frontend's Node process gets its own service unit in **Phase 8**, once its build exists — same reasoning applied earlier to Nuitka-before-systemd, just one level up the stack: don't wire a service to something that isn't built yet.

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

`frontend.env` is added to the backup list (small, config-like, same as `.env`) and `frontend/` is added to the "never backup" list (it's a build artifact, same treatment as `backend/`).

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

Password-protection notes: use `zip -e` for cron, or 7-Zip for Windows if `Compress-Archive`'s lack of encryption is a concern.

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
# Phase 8 — Frontend Hosted on the Server + Tauri Thin Client

Three things happen in this phase, in order: the frontend gets built for **server** hosting instead of per-client bundling; it gets wrapped in its own OS service and unified with the backend behind Nginx; and the Tauri shell is built from scratch as a "thin, licensed window onto the UI," never as a container that ships the UI itself.

## Task 8.1 — Why Runtime Config for the Frontend Itself Is No Longer Needed

The reason `NEXT_PUBLIC_*` doesn't work here is that it's inlined at build time, and every client PC might need a different backend address, so one build can't serve every machine.

That constraint is gone here, because the frontend is no longer built once per client — **it's built exactly once, for exactly one target: the server it's about to run on.** It doesn't need to know a LAN IP at all, at build time or run time, because it talks to the backend through a relative path (`/api/...`) that resolves against whatever address the browser (or Tauri's webview) used to load the page in the first place — see Task 8.4. There is nothing analogous to `config.env` needed on the frontend side.

The runtime-config problem doesn't vanish, though — it moves to the Tauri shell, which still doesn't know the server's LAN address until install time (Task 8.5).

## Task 8.2 — Build the Frontend for Server Hosting

Use Next.js's `standalone` output instead of `export` — the server can run a real Node process, so there's no reason to give up SSR/server actions/API routes just to avoid needing Node on every client PC.

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

With this in place, `http://192.168.1.100/` serves the app UI and `http://192.168.1.100/api/...` serves the API, both from the same origin. That's what eliminates the CORS requirement for browser-side calls (Task 2.4) and gives the Tauri shell exactly one address to remember (Task 8.5), instead of one for the UI and one for the API.

## Task 8.5 — The Tauri Shell

The Tauri app is a small Rust binary plus a system webview — it never loads a bundled Next.js export. It still does a pre-flight license check first, then points a native window at the server instead of at local content, exactly like the Electron version of this plan would have, but implemented through Tauri's Rust core instead of an Electron main process, and through Tauri's capability-scoped IPC instead of a preload/contextBridge pair.

### Project layout

```text
app-client/
├── src-tauri/
│   ├── src/
│   │   └── main.rs           # Rust core — equivalent of Electron's main.js
│   ├── capabilities/
│   │   └── main.json         # scopes which windows/origins may call which commands
│   ├── icons/
│   └── tauri.conf.json
├── dist/                      # minimal LOCAL frontend — fallback screen only, no app UI
│   └── license-invalid.html
└── package.json                # dev-only, for the @tauri-apps/cli
```

`dist/` is what Tauri calls its bundled "frontend" — but here it contains nothing but the fallback screen. The real application UI is never part of this project; it's fetched live from `SERVER_URL`, same as in the Electron version.

### Runtime config: `config.env` in the app's config directory

Tauri resolves a per-user, per-app config directory itself (via the `identifier` in `tauri.conf.json`, e.g. `com.yourcompany.appclient`) — writable without admin rights, survives app updates, and lives outside the installed binary. On Windows this lands under `%APPDATA%\com.yourcompany.appclient\`, mirroring where `userData` would have put it under Electron.

```env
# Single LAN address for the whole app — frontend UI and API both live behind this one origin
SERVER_URL=http://192.168.1.100

# Optional feature flags the Tauri shell itself reads (most flags now live in the frontend instead)
ENABLE_TELEMETRY=false
```

### Rust core (`src-tauri/src/main.rs`)

```rust
// src-tauri/src/main.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::fs;
use std::path::PathBuf;
use std::time::Duration;
use tauri::{Emitter, Manager, WebviewUrl, WebviewWindowBuilder};

#[derive(Serialize, Deserialize, Clone, Default)]
struct LicenseStatus {
    valid: bool,
    reason: Option<String>,
}

fn config_path(app: &tauri::AppHandle) -> PathBuf {
    let dir = app.path().app_config_dir().expect("no app config dir");
    fs::create_dir_all(&dir).ok();
    dir.join("config.env")
}

fn load_or_create_config(app: &tauri::AppHandle) -> HashMap<String, String> {
    let path = config_path(app);
    if !path.exists() {
        // Copy the bundled default.env resource, or fall back to a hardcoded default —
        // same idea as Electron reading extraResources/default.env on first launch.
        match app
            .path()
            .resolve("resources/default.env", tauri::path::BaseDirectory::Resource)
        {
            Ok(resource_path) if resource_path.exists() => {
                let _ = fs::copy(&resource_path, &path);
            }
            _ => {
                let _ = fs::write(&path, "SERVER_URL=http://localhost\n");
            }
        }
    }
    fs::read_to_string(&path)
        .unwrap_or_default()
        .lines()
        .filter_map(|l| l.split_once('='))
        .map(|(k, v)| (k.trim().to_string(), v.trim().to_string()))
        .collect()
}

#[tauri::command]
fn get_config(app: tauri::AppHandle) -> HashMap<String, String> {
    load_or_create_config(&app)
}

#[tauri::command]
fn set_config(app: tauri::AppHandle, new_values: HashMap<String, String>) -> HashMap<String, String> {
    let mut merged = load_or_create_config(&app);
    merged.extend(new_values);
    let serialized: String = merged
        .iter()
        .map(|(k, v)| format!("{k}={v}"))
        .collect::<Vec<_>>()
        .join("\n");
    let _ = fs::write(config_path(&app), serialized + "\n");
    merged
}

async fn check_server_license(server_url: &str) -> LicenseStatus {
    let client = match reqwest::Client::builder().timeout(Duration::from_secs(5)).build() {
        Ok(c) => c,
        Err(_) => return LicenseStatus { valid: false, reason: Some("unreachable".into()) },
    };
    match client.get(format!("{server_url}/api/license/status/")).send().await {
        Ok(res) if res.status().is_success() => res
            .json::<LicenseStatus>()
            .await
            .unwrap_or(LicenseStatus { valid: false, reason: Some("http_error".into()) }),
        Ok(_) => LicenseStatus { valid: false, reason: Some("http_error".into()) },
        Err(_) => LicenseStatus { valid: false, reason: Some("unreachable".into()) },
    }
}

fn show_fallback(app: &tauri::AppHandle, reason: &str) {
    let win = WebviewWindowBuilder::new(app, "fallback", WebviewUrl::App("license-invalid.html".into()))
        .inner_size(480.0, 320.0)
        .resizable(false)
        .build()
        .expect("failed to build fallback window");

    // Fallback content is LOCAL (bundled in dist/), so it's free to use the full
    // Tauri JS API — unlike the main window below, it never needs the `remote`
    // capability scoping because it isn't remote content.
    let reason = reason.to_string();
    win.on_page_load(move |webview, _payload| {
        let _ = webview.emit("license-error", LicenseStatus { valid: false, reason: Some(reason.clone()) });
    });
}

fn open_main_window(app: &tauri::AppHandle, server_url: &str) {
    let parsed = url::Url::parse(server_url).expect("invalid SERVER_URL");
    let allowed_origin = format!("{}://{}", parsed.scheme(), parsed.host_str().unwrap_or(""));

    let _win = WebviewWindowBuilder::new(app, "main", WebviewUrl::External(parsed))
        .inner_size(1280.0, 800.0)
        // NEW — lock navigation to the configured server origin. Content is remote,
        // so nothing else stops the loaded page from trying to carry the shell
        // elsewhere unless this guard is here — the direct equivalent of Electron's
        // will-navigate handler.
        .on_navigation(move |url| {
            format!("{}://{}", url.scheme(), url.host_str().unwrap_or("")) == allowed_origin
        })
        .build()
        .expect("failed to build main window");
}

// NEW — Tauri has no exact equivalent of Electron's did-fail-load event for a
// connectivity drop mid-session. Instead, the frontend runs a small heartbeat
// (Task 8.6) and calls this command if a health check fails, which closes the
// main window and shows the same fallback screen.
#[tauri::command]
fn report_connection_lost(app: tauri::AppHandle, reason: String) {
    if let Some(w) = app.get_webview_window("main") {
        let _ = w.close();
    }
    show_fallback(&app, &reason);
}

#[tauri::command]
async fn retry_license_check(app: tauri::AppHandle) -> LicenseStatus {
    let cfg = load_or_create_config(&app);
    let server_url = cfg.get("SERVER_URL").cloned().unwrap_or_else(|| "http://localhost".into());
    let status = check_server_license(&server_url).await;
    if status.valid {
        for (_, w) in app.webview_windows() {
            let _ = w.close();
        }
        open_main_window(&app, &server_url);
    }
    status
}

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            get_config,
            set_config,
            retry_license_check,
            report_connection_lost
        ])
        .setup(|app| {
            let handle = app.handle().clone();
            tauri::async_runtime::block_on(async move {
                let cfg = load_or_create_config(&handle);
                let server_url = cfg.get("SERVER_URL").cloned().unwrap_or_else(|| "http://localhost".into());
                let status = check_server_license(&server_url).await;
                if status.valid {
                    open_main_window(&handle, &server_url);
                } else {
                    show_fallback(&handle, status.reason.as_deref().unwrap_or("unknown"));
                }
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

The pre-flight license check runs before any application window opens, calls `/api/license/status/`, and shows a local fallback screen on failure — same sequencing as the Electron version's `startApp()`. Beyond that, this task adds: the `on_navigation` guard, the `report_connection_lost` command standing in for `did-fail-load`, and `WebviewUrl::External` in place of loading a local file.

### Capabilities — the preload/contextBridge replacement

Electron's `contextBridge` works for remote content automatically, because a preload script attaches to a `BrowserWindow` regardless of what origin it later navigates to. Tauri's IPC is origin-aware by design: a webview showing **remote** content only gets access to Tauri commands if a capability file explicitly says so, via a `remote.urls` allow-list. Since `SERVER_URL` is whatever LAN address the operator configures at install time (Task 8.5's `config.env`), the pattern has to cover your private network ranges rather than one fixed domain — scope it as tightly as your deployment allows (a stable hostname like `http://app-server.local/*` is safer than wildcarding every private range, if you can arrange for that hostname to resolve on the LAN):

```json
// src-tauri/capabilities/main.json
{
  "identifier": "main-capability",
  "windows": ["main"],
  "remote": {
    "urls": [
      "http://192.168.*.*/*",
      "http://10.*.*.*/*",
      "http://172.16.*.*/*",
      "http://172.17.*.*/*",
      "http://172.31.*.*/*",
      "http://localhost/*"
    ]
  },
  "permissions": [
    "core:event:allow-listen",
    "core:event:allow-emit"
  ]
}
```

```json
// src-tauri/capabilities/fallback.json — local content, no remote scoping needed
{
  "identifier": "fallback-capability",
  "windows": ["fallback"],
  "permissions": [
    "core:event:allow-listen",
    "core:event:allow-emit"
  ]
}
```

Custom commands (`get_config`, `set_config`, `retry_license_check`, `report_connection_lost`) are exposed to a window's capability set the same way — add each command's auto-generated permission identifier (Tauri generates one per `#[tauri::command]`, following the `<plugin-or-app>:allow-<command_name>` convention) to the relevant capability's `permissions` array. Keep the `main` window's capability limited to only the handful of commands the Settings screen (Task 8.6) actually needs — resist the temptation to grant it `core:default`, which is far broader than remote content should ever get.

## Task 8.6 — In-App Settings Screen

```ts
// Settings page, running inside the loaded frontend, calling back into the Tauri shell
import { invoke } from '@tauri-apps/api/core';

await invoke('set_config', { newValues: { SERVER_URL: 'http://10.0.0.5' } });
// Then prompt: "Restart now to apply changes?"
```

This screen is served by the frontend (part of the normal app UI), not bundled Tauri-side code — but it still needs to call back into the shell, since only the Rust core can write `config.env` in the app's config directory. Two things make that call possible for remotely-loaded content specifically:

1. Enable `app.withGlobalTauri: true` in `tauri.conf.json` so the Tauri JS API is injected as a `window.__TAURI__` global into any page the webview loads — bundled or remote — without the frontend project needing `@tauri-apps/api` as an actual npm dependency. (If you'd rather have proper types and don't mind adding the dependency to the Next.js project, importing `@tauri-apps/api/core` directly works too, and is what the snippet above shows.)
2. The page's origin must match one of the `remote.urls` patterns in Task 8.5's capability file, and the specific command being called must be in that capability's `permissions` list — this is the part that has no Electron equivalent to fall back on; it's Tauri's actual security boundary, not an optional hardening step.

The same origin-lock from Task 8.5's `on_navigation` guard is what makes this safe to leave enabled: nothing the shell could load other than your own server's pages can reach these commands.

### The heartbeat (fills the did-fail-load gap)

Since Tauri doesn't expose a "the loaded page failed to load" event the way Electron's `did-fail-load` does, add a small heartbeat to the frontend itself so a mid-session outage still produces the same fallback experience:

```ts
// runs inside the frontend, main window only
import { invoke } from '@tauri-apps/api/core';

setInterval(async () => {
  try {
    const res = await fetch('/api/license/status/', { signal: AbortSignal.timeout(5000) });
    if (!res.ok) throw new Error('unhealthy');
  } catch {
    await invoke('report_connection_lost', { reason: 'unreachable' });
  }
}, 15000);
```

## Task 8.7 — Package With the Tauri CLI

```bash
npm install --save-dev @tauri-apps/cli
```

```json
// src-tauri/tauri.conf.json
{
  "productName": "App Client",
  "identifier": "com.yourcompany.appclient",
  "build": {
    "frontendDist": "../dist"
  },
  "app": {
    "withGlobalTauri": true,
    "windows": []
  },
  "bundle": {
    "active": true,
    "targets": ["nsis"],
    "resources": ["resources/default.env"],
    "icon": ["icons/icon.ico"],
    "windows": {
      "nsis": {
        "installMode": "currentUser"
      }
    }
  }
}
```

```bash
npm run tauri build
```

`installMode: "currentUser"` is the closest Tauri equivalent to electron-builder's `perMachine: false` — a per-user install with no admin prompt. Tauri's NSIS bundler doesn't expose a literal one-click flag the way electron-builder does; if a truly zero-dialog install is a hard requirement, that means customizing the bundler's NSIS template rather than flipping a config key.

Note there's no Next.js output referenced anywhere in `tauri.conf.json` — `frontendDist` points at the tiny local `dist/` folder containing only the fallback screen, so the resulting installer stays a fraction of the size a bundled application UI would add. Tauri also isn't shipping its own copy of a browser engine the way Electron does, which shrinks things further on top of that.

## Task 8.8 — Test the Packaged App

```text
✓ App installs without an admin prompt
✓ App starts, checks the license against SERVER_URL, and loads the app UI live from the server
✓ Editing config.env (via the Settings screen or by hand) + restarting picks up the new SERVER_URL
✓ Logs land in %APPDATA%\com.yourcompany.appclient\logs\ (or wherever your logging setup writes them)
✓ License-invalid screen shows the right message for each failure reason
✓ Retry button re-checks and opens the main window once the license is valid again
✓ Stopping the frontend service (app-frontend.service) while the shell is open triggers the
  heartbeat's report_connection_lost fallback within one interval, not a blank/frozen window
✓ Attempting to navigate the loaded page to a different origin is blocked by on_navigation
✓ Calling a Tauri command from an origin NOT in remote.urls is rejected — verified by
  temporarily pointing SERVER_URL somewhere outside the allow-listed ranges
```

---

# STOP

```text
✓ Frontend builds once, runs as its own service on the server, and is reachable through Nginx
✓ Nginx unifies frontend + API behind one origin — no CORS preflight for browser-side calls
✓ Tauri app packaged as a per-user installer, with no bundled application UI inside it
✓ Runtime config.env editable post-build via the Settings screen or by hand, holding SERVER_URL
✓ App reaches the real, compiled, serviced backend AND the real, serviced frontend on the LAN
✓ license-invalid.html exists and renders correctly, with a working retry path
✓ Navigation guard and the heartbeat-driven connection-lost fallback both verified working
✓ capabilities/main.json scopes remote IPC to only the commands and origins actually needed
```

---

<a id="phase-9"></a>
# Phase 9 — Frontend Licensing Gate in Tauri

The Tauri shell calls `/api/license/status/` **before** opening any application window (already wired in Phase 8's `setup()` closure). If invalid, it shows the local fallback with the reason and a working retry button instead of the app.

The remotely-loaded page doesn't need to know about licensing at all — if the main window opened, the license was valid at that moment. Backend API calls still return 401/403 if the license is revoked mid-session, handled by showing a generic "session expired" dialog in the page.

**One genuine improvement from centralizing the frontend:** there is now exactly one deployment of the app UI to keep in sync with license state, instead of one per client PC. Fixing a licensing-adjacent UI bug (say, a wrong message on the renew screen) means redeploying `app-server/frontend/` once — every Tauri shell picks it up automatically on its next load, with no redistribution step.

## Task 9.1 — Full-Stack Lifecycle Test

```text
✓ valid license          → main window opens, app runs normally
✓ missing license file    → fallback window with reason="missing"
✓ expired license         → fallback window with reason="expired"
✓ license for wrong hwid  → fallback window with reason="hwid_mismatch"
✓ tampered license file   → fallback window with reason="tampered"
✓ backend unreachable      → fallback window with reason="unreachable"
✓ frontend service down (backend fine) → heartbeat fires report_connection_lost, reason="unreachable"
✓ (online mode) revoked   → blocked after grace period, all connected clients reflect it
✓ license revoked mid-session → next API call returns 403, page shows session-expired dialog
✓ fixing the license server-side, then clicking Retry → app opens without a restart
```

## Task 9.2 — Multi-Client Test

Install the Tauri shell on **3 different client PCs** on the same LAN, all pointing at the same `SERVER_URL`:

```text
✓ All three can log in and use the app concurrently, all served from the same frontend build
✓ If max_clients is set in the license and exceeded, the 4th client gets a clear "client limit reached" message
✓ Restarting the frontend service on the server briefly disconnects all three clients the same way,
  and all three recover independently via their own retry paths — no client-specific state to fix
```

---

# STOP

```text
✓ Tauri client reflects license status correctly for every backend AND frontend state
✓ Multiple client PCs work concurrently against the same server-hosted frontend
```

---

<a id="phase-10"></a>
# Phase 10 — Two Installers

## Server installer (`app-server-setup.exe`)

Bundles the compiled backend **plus** the frontend build and its service registration:
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

Produced by the Tauri CLI (`npm run tauri build`) in Phase 8. This is where the biggest practical change shows up: it bundles **only the Tauri shell** — no Next.js build, no renderer bundle, just:
- The Tauri binary (Rust core + system webview), the fallback HTML, and the capability config baked in at build time.
- `default.env` template (copied to the app's config directory as `config.env` on first launch).

On first run, the app:
1. Asks the user for the **server's LAN address** (or auto-discovers via mDNS, optional).
2. Writes `SERVER_URL=http://<server-ip>` to `config.env` via the `set_config` command.
3. Restarts and connects — at which point it's loading the exact same live UI every other client on the LAN is loading.

Still one-click from the user's perspective: install → enter server address → use. The installer itself is dramatically smaller, since it isn't shipping an application UI — or a bundled browser engine — at all.

## Distribution

- Server installer: delivered once to IT (or installed by you on-site) — now sets up two services instead of one, plus Nginx.
- Client installer: distributed via shared network drive, USB stick, or internal download link — just a much smaller file than a full application bundle would be. Frontend updates no longer require touching this installer at all — only backend or Tauri-shell changes do.

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

Most of this was implemented earlier under "do this now, not at the end." A few items are specifically here because content is now loaded from the network rather than bundled locally.

### Backend (server)
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
- **HTTPS matters even more in this variant.** Not just API traffic crosses the network in the clear — the entire application UI does too, on every page load. If the LAN spans untrusted segments (guest Wi-Fi, shared switches, anything outside a controlled closet), terminate TLS at Nginx.
- CORS: `CORS_ALLOWED_ORIGINS` locked to the Nginx origin, never `"*"` — this now doubles as a backstop for direct calls to the (closed) backend port rather than the primary defense.

### Frontend (client PCs) — updated for this variant
- **No Node/OS API surface is exposed to the webview by default.** Unlike Electron, where `contextIsolation`/`nodeIntegration` have to be *configured* correctly, Tauri's webview starts with zero access to the Rust side — a page only gets a command if that command is both registered in `invoke_handler` and explicitly granted to that window's capability. The hardening work here is auditing `capabilities/main.json` (Task 8.5), not flipping isolation flags on.
- Confirm the `main` window's capability grants **only** `get_config` / `set_config` / `retry_license_check` / `report_connection_lost` — not `core:default`, which is far broader than remotely-loaded content should ever receive.
- Confirm `remote.urls` patterns are scoped to your actual private network ranges, never a bare wildcard (`*` or `http://*/*`) — a wildcard here defeats the whole point of origin-scoped IPC.
- **CSP moves server-side.** Tauri's built-in `app.security.csp` setting in `tauri.conf.json` only injects a `<meta>` tag into locally-bundled content (the fallback screen) — it has no effect on the remotely-loaded main window, since the webview doesn't own that page's markup any more than Electron's would. Set the CSP as a real HTTP response header from Nginx or Next.js instead:
  ```nginx
  add_header Content-Security-Policy "default-src 'self'; connect-src 'self'; frame-ancestors 'none';" always;
  ```
- **Navigation is locked to the server's origin.** Implemented in Task 8.5 via the `on_navigation` closure on the main `WebviewWindowBuilder`. Since content is remote, it has somewhere untrusted to navigate to, so this guard is mandatory, not optional.
- **The frontend Node process binds to `127.0.0.1`**, never `0.0.0.0` — it should be unreachable except through Nginx on the same machine, exactly like the backend.
- Don't log `config.env` contents to disk. `config.env` carries relatively little (`SERVER_URL` plus a couple of flags, no API keys duplicated), which limits the blast radius of a leaked copy, but the rule still stands.
- If using the Tauri updater plugin (`tauri-plugin-updater`): updates are signature-verified automatically against the public key configured in `tauri.conf.json`'s `plugins.updater.pubkey` — generate the keypair once with `tauri signer generate` and keep the private half at least as protected as the license-signing key from Phase 4.
- `config.env` in the app's config directory is readable by that OS user only by default — don't weaken it.

---

<a id="milestone-order"></a>
# Milestone Order

1. **Phase 1–2** — backend runs correctly, uncompiled, reachable from another machine, CORS/ALLOWED_HOSTS driven by `.env`, Nginx serving static/media/API (no frontend block yet).
2. **Phase 3** — logging wired up for the backend.
3. **Phase 4** — backend licensing enforced and fully tested, still uncompiled, HWID hand-off defined.
4. **Phase 5** — compile the backend with Nuitka, re-test licensing against the compiled binary.
5. **Phase 6** — wrap the *compiled backend* in a systemd/NSSM service.
6. **Phase 7** — backups wired around the live, serviced backend.
7. **Phase 8** — frontend built for server hosting, wrapped in its own service, unified with the backend behind Nginx, and the Tauri shell built as a thin client pointing at that unified origin, with its license-invalid screen, retry path, navigation guard, and heartbeat-driven connection-lost fallback all functional. **Server track and frontend build both finish here.**
8. **Phase 9** — frontend license gate re-verified end-to-end, multi-client test across several PCs against the one shared frontend.
9. **Phase 10** — two separate installers produced and tested end-to-end (server installer now sets up two services; client installer ships no application UI at all).
10. **Phase 11** — hardening audit across backend, frontend, and the Tauri shell, including closing the raw 8000/3000 ports, moving CSP server-side, and auditing the capability/remote-origin scoping.

Each phase's STOP checkpoint should pass before starting the next one. Backend and frontend remain two independently deployable services on the server, with their own config files and their own service units, unified only at the Nginx layer — the Tauri shell's config is editable post-build via `config.env` in the app's config directory + the `get_config`/`set_config` commands, and now holds a single `SERVER_URL` rather than a separate API address.

---

<a id="future-feature"></a>
# Future Feature — License Renewal Endpoint

The mechanism here applies to any project built on this template. The one adjustment worth calling out is where the renewal *UI* now lives.

## The problem

Renewing an expired or changed license today means generate a new `license.lic` → send it to the customer/IT → someone manually overwrites `config/license.lic` on the server → restart the service.

## Two ways to solve it

### A. Push model — an authenticated renewal endpoint

Add `POST /api/license/renew/` that accepts a new signed token and applies it live, without a manual file swap or service restart.

**Design constraints:**
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

from src.core.license_validator import LicenseValidator

logger = logging.getLogger("license")

@api_view(["POST"])
@permission_classes([IsAdminUser])
def renew_license(request):
    new_token = request.data.get("license_token", "").strip()
    if not new_token:
        return Response({"valid": False, "reason": "missing_token"}, status=400)

    validator = LicenseValidator.from_file(os.path.join(CONFIG_DIR, "public_key.pem"))
    try:
        payload = validator.validate(new_token)  # checks signature, expiry, AND hwid in one call
    except ValueError as e:
        return Response({"valid": False, "reason": str(e)}, status=400)

    try:
        current = validator.validate_from_file(os.path.join(CONFIG_DIR, "license.lic"))
        if current.license_id != payload.license_id and not request.data.get("force"):
            return Response({"valid": False, "reason": "license_id_mismatch"}, status=400)
    except ValueError:
        pass  # no valid current license to compare against — allow the renewal through

    license_path = os.path.join(CONFIG_DIR, "license.lic")
    fd, tmp_path = tempfile.mkstemp(dir=CONFIG_DIR)
    with os.fdopen(fd, "w") as f:
        f.write(new_token)
    os.replace(tmp_path, license_path)
    os.chmod(license_path, 0o600)

    logger.info(f"License renewed via API: {payload.license_id} exp={payload.exp}")
    return Response({
        "valid": True,
        "client": payload.client,
        "exp": payload.exp,
        "features": payload.features,
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

- A scheduled job calls `payload["server_url"]` with `license_id` + `hwid` on a daily interval.
- A newer signed token runs through the same validation and atomic-write path as the push endpoint, factored into a shared `apply_new_license(token)` function.
- Unreachable license server falls back to `grace_days` handling.

```python
# license_renewal.py
from src.core.license_validator import LicenseValidator

def apply_new_license(token, config_dir=CONFIG_DIR):
    """Shared by both the push endpoint and the pull/cron job."""
    validator = LicenseValidator.from_file(os.path.join(config_dir, "public_key.pem"))
    payload = validator.validate(token)  # raises ValueError (e.g. "hwid_mismatch") if invalid

    license_path = os.path.join(config_dir, "license.lic")
    fd, tmp_path = tempfile.mkstemp(dir=config_dir)
    with os.fdopen(fd, "w") as f:
        f.write(token)
    os.replace(tmp_path, license_path)
    os.chmod(license_path, 0o600)
    logger.info(f"License renewed: {payload.license_id} exp={payload.exp}")
    return payload
```

## Frontend Hook-up

The "Renew License" screen is a normal page in the one, centrally-hosted frontend (Task 8.2) — implemented once, not once per client. It calls the renewal endpoint the same way any other page calls the API (relative `fetch('/api/license/renew/')`, same-origin through Nginx):

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

The Tauri shell's `retry_license_check` command still exists (Task 8.5) and is still useful — if a fallback window happens to be showing on a given client PC when the license gets renewed, that client's Retry button will pick up the change on the next click. The renew action itself doesn't need to know it's running inside Tauri at all; it's ordinary web app code, and it fixes the state for every connected client the moment it runs, not just the one that clicked it.

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
✓ a Tauri shell showing the fallback window at the time of renewal recovers via its existing Retry button
```
