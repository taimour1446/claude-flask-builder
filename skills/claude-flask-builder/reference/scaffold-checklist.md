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
- utils/Settings.py — validated env config (FAIL FAST on missing required)
- utils/Logging.py — structured logger config
- utils/BaseResponse.py — envelope + redaction-aware
- utils/Auth.py — JWT generate + token_required + role_required + refresh_token
- utils/RequestInterceptor.py — X-Request-ID + redaction
- utils/Validation.py — universal validate w/ unknown=EXCLUDE
- utils/Helper.py — pagination, secrets-based generators
- utils/TimezoneHelper.py — pytz/zoneinfo wrappers
- utils/Constants.py — all enums + error message classes

### 4. Configuration
- configuration/Database.py, Mail.py, Scheduler.py (correct spelling)
- configuration/Configuration.py — orchestrator (NO db.create_all)

### 5. Model: base auth
- model/User.py with: id, email (UNIQUE), phone (UNIQUE), _password (hybrid+bcrypt), role,
  ptoken + ptoken_expires_at, otp + otp_created_at + otp_attempts,
  device_token, deleted_at, created/updated_at tz-aware

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
