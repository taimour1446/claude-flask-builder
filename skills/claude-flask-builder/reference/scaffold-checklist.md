# Scaffold checklist

## Inputs (ask user — cannot guess)
- App display name
- Python module name (snake_case)
- DB choice: postgres (default) / mysql
- Integrations to enable: stripe, twilio, fcm, s3, gcs, mail (each y/n)
- Target directory

## Steps (flask-scaffolder follows in order)

### 1. Project init
- `pipenv --python 3.11`
- pipenv install: flask, sqlalchemy, flask-sqlalchemy, flask-migrate, flask-seeder, marshmallow, pyjwt, bcrypt, flask-cors, flask-mail, flask-apscheduler, python-json-logger, sentry-sdk[flask] (optional), gunicorn, python-dotenv
- pipenv install --dev: pytest, pytest-flask, pytest-cov, responses, ruff, black, isort, pip-audit

### 2. Folder skeleton (see architecture.md)
Create all dirs + `__init__.py` files. The `__init__.py` of each layer is a
**barrel** — explicit re-exports, never `from .X import *`. Concrete shapes:

```python
# app/api/__init__.py — controllers expose blueprint factories
from app.api.AccountController import account_blueprint
# from app.api.<X>Controller import <x>_blueprint  ← add one line per controller
```

```python
# app/domain/service/__init__.py — services exposed for the cron jobs + DI
from app.domain.service.AccountService import AccountService
# from app.domain.service.<X>Service import <X>Service
```

```python
# app/domain/validation/__init__.py — explicit re-export of every schema class
from app.domain.validation.AccountValidation import (
    LoginValidation, SignupValidation, ForgotPasswordValidation,
    ResetPasswordValidation, SendOTPValidation, VerifyOTPValidation,
)
# from app.domain.validation.<X>Validation import Create<X>Validation, Update<X>Validation
```

```python
# app/domain/dto/__init__.py
from app.domain.dto.AccountSchema import AccountSchema
# from app.domain.dto.<X>Schema import <X>Schema
```

```python
# app/model/__init__.py
from app.model.User import User
# from app.model.<X> import <X>
```

```python
# app/scheduler_jobs/__init__.py — REQUIRED: import each job module so its
# @scheduler.task decorator runs at app boot. Without this, no cron job
# registers, no matter how many files live in the folder.
from app.scheduler_jobs import (  # noqa: F401 — imports register decorators
    # purge_pending_orders_job,
    # send_pending_otp_reminders_job,
)
```

`configuration.py.md`'s `register_blueprints` imports from `app.api`, which
relies on `app/api/__init__.py` re-exporting every blueprint factory. If you
add a new Controller, you MUST add its line to `app/api/__init__.py` AND
register it in `Configuration.register_blueprints`. The reviewer PRE-FAILs
plans that add a controller without both updates.

### 3. Foundations
Each file below has a **full skeleton in `reference/patterns.md` sections
1, 7, 8, 10, 11**. Scaffolder synthesizes from those skeletons; do NOT
invent. Cross-references after each line point to the authoritative section.

- **utils/Settings.py** — validated env config (FAIL FAST on missing required).
  Class with class-level `os.environ`-driven attrs; `Settings.validate()`
  raises on first missing required at startup. Shape: a singleton instance
  `settings = Settings()` exposed at module level; `Settings.validate()`
  called from `Configuration.configure_settings(app)`. (See `security.md`
  "What the scaffolder writes by default — Settings".)

  **Required attribute surface** (every consumer in patterns.md + templates
  depends on these — Settings MUST declare each, with a sane default where
  marked):
  | Attr | Required | Source env var | Used by |
  | --- | --- | --- | --- |
  | `APP_NAME` | yes | `APP_NAME` | integration-client User-Agent |
  | `APP_VERSION` | yes | `APP_VERSION` (semver) | health endpoint, integration-client UA |
  | `ENV` | yes | `ENV` (dev/staging/production) | BaseResponse trace gate, cookie hardening |
  | `LOG_LEVEL` | default INFO | `LOG_LEVEL` | configure_logging |
  | `SECRET_KEY` | yes | `SECRET_KEY` | Flask sessions, JWT signing |
  | `DATABASE_URI` | yes | `DATABASE_URI` | SQLAlchemy |
  | `JWT_ACCESS_TTL_MINUTES` | default 30 | `JWT_ACCESS_TTL_MINUTES` | Auth.generate_tokens |
  | `JWT_REFRESH_TTL_DAYS` | default 30 | `JWT_REFRESH_TTL_DAYS` | Auth.generate_tokens |
  | `CORS_ORIGINS` | yes | `CORS_ORIGINS` (csv → list) | Flask-CORS |
  | `MAX_CONTENT_LENGTH_BYTES` | default 8 MiB | `MAX_CONTENT_LENGTH_BYTES` | R63 upload cap |
  | `MAIL_SERVER`/`MAIL_PORT`/`MAIL_USERNAME`/`MAIL_PASSWORD`/`MAIL_DEFAULT_SENDER` | when mail toggled | corresponding env | Flask-Mail |
  | `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` / `STRIPE_WEBHOOK_SECRET` / `STRIPE_RETURN_URL_ALLOWLIST` (csv → list) | when stripe toggled | corresponding env | Stripe integration |
  | `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_FROM_NUMBER` | when twilio toggled | corresponding env | Twilio integration |
  | `AWS_ACCESS_KEY` / `AWS_ACCESS_SECRET` / `S3_BUCKET` | when s3 toggled | corresponding env | S3 integration |

  `Settings.validate()` enforces: required attrs are non-empty;
  `len(SECRET_KEY.encode("utf-8")) >= 32`; `ENV in {"dev","staging","production"}`;
  `CORS_ORIGINS` is a non-empty list.

  **CSV-shaped env vars** (currently `CORS_ORIGINS` and
  `STRIPE_RETURN_URL_ALLOWLIST`) are read as comma-separated strings from
  the environment and TRANSFORMED into `list[str]` at Settings instantiation.
  Settings exposes them as `list[str]`; downstream callers (Flask-CORS,
  Stripe integration) iterate them. The canonical helper:
  ```python
  def _csv(name: str, default: str = "") -> list[str]:
      raw = os.environ.get(name, default)
      return [v.strip() for v in raw.split(",") if v.strip()]
  ```
  Full Settings skeleton lives in `patterns.md §12` — do not invent.

  **`configure_settings(app)` MUST mirror critical env values into `app.config`**:
  at minimum `app.config["ENV"] = settings.ENV`, `app.config["SECRET_KEY"]`,
  `app.config["SESSION_COOKIE_*"]` flags (R74 — `SECURE` MUST be
  `settings.ENV == "production"`; `HTTPONLY=True`; `SAMESITE="Lax"`), and
  `app.config["MAX_CONTENT_LENGTH"] = settings.MAX_CONTENT_LENGTH_BYTES`
  (R63). This is what lets `BaseResponse.respondError` gate its `trace`
  field on production (`current_app.config.get("ENV") != "production"`).

  **`SECRET_KEY`** must be `secrets.token_urlsafe(32)` (≥256 bits) — Settings
  rejects a SECRET_KEY shorter than 32 raw bytes at startup. The placeholder
  in `.env.example` is intentionally invalid so deploys fail fast.
- **utils/Logging.py** — `configure_logging()` installs JSON formatter from
  `python-json-logger` when `ENV in {staging, production}`; plain stdlib
  formatter in dev. Root level driven by `LOG_LEVEL` env.
- **utils/BaseResponse.py** — see `patterns.md §1` (FULL SKELETON).
- **utils/Auth.py** — see `patterns.md §7` (FULL SKELETON: `generate_tokens`,
  `assert_role`, `token_required` decorator).
- **utils/RequestInterceptor.py** — see `patterns.md §11` (FULL SKELETON
  with redaction allowlist).
- **utils/Validation.py** — see `patterns.md §10` (FULL SKELETON with
  `CustomValidationException`).
- **utils/Helper.py** — at minimum: `pagination(schema, data, query)`
  (returns `{pagination: {...}, list: [...]}`), `generate_otp()` using
  `secrets.randbelow`, `generate_ptoken()` using `secrets.token_urlsafe(32)`,
  `merge_dict(base, override)`. NEVER `random.*` for any of these (R20).
- **utils/TimezoneHelper.py** — `to_utc(naive, tz_name)`, `from_utc(aware,
  tz_name)` via `zoneinfo.ZoneInfo` (stdlib, no pytz dep).
- **utils/Constants.py** — every enum + error-message class. Use stdlib
  `enum.Enum` for value enums; plain `class NameTaken: message = "..."` for
  error-message catalogs (the values get passed to
  `CustomValidationException`).

### 4. Configuration
- configuration/Database.py, Mail.py, Scheduler.py (correct spelling)
- configuration/Configuration.py — orchestrator (NO db.create_all)

### 5. Model: base auth
- model/User.py with: id, email (UNIQUE), phone (UNIQUE), _password (hybrid+bcrypt), role,
  ptoken + ptoken_expires_at, otp + otp_created_at + otp_attempts,
  device_token, refresh_jti (String(32), nullable, set to `secrets.token_urlsafe(16)` (~22 chars) on each refresh — required by `Auth.generate_tokens` in `patterns.md §7`; the column size headroom (32 vs 22) is intentional so swapping the jti generator to `token_urlsafe(24)` (~32 chars) does not require a migration),
  deleted_at, created/updated_at tz-aware

### 6. API + service + validation + DTO for auth
- AccountController + AccountService + AccountSchema
- Validation classes (one file `AccountValidation.py`, many classes):
  `LoginValidation`, `SignupValidation`, `ForgotPasswordValidation`,
  `ResetPasswordValidation`, `SendOTPValidation`, `VerifyOTPValidation`
- Endpoints (all under `/api/v1`):
  `POST /auth/login`, `POST /auth/signup`, `POST /auth/forgot-password`,
  `POST /auth/reset-password`, `POST /auth/send-otp`, `POST /auth/verify-otp`,
  `POST /auth/refresh`, `GET /auth/me` (protected by `@token_required`)
- The full controller + companion service shape lives in
  `templates/auth-controller.py.md`. Scaffolder uses it verbatim.

### 7. App factory
- application.py per architecture.md initialization order

### 8. Migrations
- `flask db init` — fix alembic.ini script_location=migrations
- Customize migrations/script.py.mako (black-format post-write hook on)
- First migration: User table + indexes (email, phone, role, deleted_at)
- env.py from Flask-Migrate standard

### 9. Seeders
- seeds/UserSeeder.py — env-driven admin creation (NO hardcoded password)
- Idempotent (check exists before insert)

### 10. Docker + CI
- Multi-stage Dockerfile (builder + runtime), non-root, HEALTHCHECK, gunicorn entrypoint
- docker-entrypoint.sh: wait-for-db, alembic upgrade, exec gunicorn
- .dockerignore comprehensive
- .github/workflows/test.yml: ruff + black + pytest with postgres service

### 11. Tests
- conftest.py with all fixtures from testing.md
- Smoke test on /health/check and /auth/login

### 12. Optional integrations (if toggled)
Each generates a stub in integrations/<provider>.py + Settings keys + .env.example entries

### 13. Verify
- `pipenv run ruff check .`
- `pipenv run black --check .`
- `pipenv run pytest --cov-fail-under=60`
- Hand to flask-pattern-reviewer
- Hand to flask-runner

## Definition of done
- App runs (gunicorn): GET /api/v1/health/check returns 200 JSON
- pytest green
- ruff+black clean
- flask-pattern-reviewer PASS
- LICENSE + comprehensive README + .env.example
