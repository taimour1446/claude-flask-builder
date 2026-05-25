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
Create all dirs + empty __init__.py files.

### 3. Foundations
Each file below has a **full skeleton in `reference/patterns.md` sections
1, 7, 8, 10, 11**. Scaffolder synthesizes from those skeletons; do NOT
invent. Cross-references after each line point to the authoritative section.

- **utils/Settings.py** — validated env config (FAIL FAST on missing required).
  Class with class-level `os.environ`-driven attrs; `Settings.validate()`
  raises on first missing required at startup. Shape: read env in
  `__init_subclass__`-like pattern OR a `@classmethod validate(cls)` called
  from `Configuration.configure_settings(app)`. (See `security.md` "What the
  scaffolder writes by default — Settings".)
  **`configure_settings(app)` MUST mirror critical env values into `app.config`**:
  at minimum `app.config["ENV"] = settings.ENV`, `app.config["SESSION_COOKIE_*"]`
  flags (R74), and `app.config["MAX_CONTENT_LENGTH"]` (R63). This is what
  lets `BaseResponse.respondError` gate its `trace` field on production
  (`current_app.config.get("ENV") != "production"`).
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
  device_token, refresh_jti (String(32), nullable, regenerated on each refresh — required by `Auth.generate_tokens` in `patterns.md §7`),
  deleted_at, created/updated_at tz-aware

### 6. API + service + validation + DTO for auth
- AccountController + AccountService + (Login/SignupValidation, ResetPasswordValidation, OTPValidation) + AccountSchema
- /api/v1/auth/login, /signup, /reset-password, /verify-otp

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
