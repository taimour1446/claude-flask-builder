# Conventions

## Naming
- Files: `XController.py`, `XService.py`, `XValidation.py`, `XSchema.py`, `X.py` (models PascalCase singular)
- Tables: plural snake_case (users, orders)
- Blueprint factory: `x_blueprint() -> Blueprint`, name='x' (snake_case)
- Route paths: kebab/snake, /api/v1 prefix mandatory, RESTful (GET=read, POST=create, PUT=update, DELETE=delete)
- Validation classes: `<Action><Resource>Validation` (CreateOrderValidation, UpdateOrderValidation)
- Multi-class-per-file ALLOWED for validation (AccountValidation has many)

## Barrels (__init__.py)
- api/__init__.py: `from .XController import x_blueprint` for each
- domain/dto/__init__.py: explicit imports of all schemas
- domain/validation/__init__.py: explicit imports of all validation classes (reference left empty — wrong)
- domain/service/__init__.py: explicit imports
- model/__init__.py: explicit imports

## i18n
- Error messages and email subjects in `app/i18n/en.json` (optional Flask-Babel scaffold)
- Constants.py holds error message strings as class attrs

## Env vars
- Settings class in utils/Settings.py reads + validates ALL env vars at startup
- .env.example documents every var with safe placeholder
- Required vars: DATABASE_URI, SECRET_KEY, BASE_URL
- Optional toggled: STRIPE_SECRET_KEY, STRIPE_PUBLISHABLE_KEY, STRIPE_WEBHOOK_SECRET, STRIPE_RETURN_URL_ALLOWLIST, TWILIO_*, FCM_SERVER_KEY, AWS_*, GCS_BUCKET, etc.

## Logging
- One named logger per module: `logger = logging.getLogger(__name__)`
- Levels: DEBUG (dev), INFO (per-request boundary), WARNING (recoverable), ERROR (caught exception), CRITICAL (data loss)
- JSON formatter in prod, colored in dev (via python-json-logger)
- NEVER `print()` in src

## EAS / Build / CI
- Multi-stage Dockerfile (builder + runtime)
- Non-root user `app:app`, uid 1000
- HEALTHCHECK curl /api/v1/health/check
- gunicorn with workers from env (default 4)
- .github/workflows/test.yml: ruff + black + pytest with postgres service, coverage --cov-fail-under=60
- Optional .github/workflows/deploy.yml — user-customized (stub points out it's project-specific)

## Git
- Author NEVER "Claude"
- Imperative commit messages
- Feature branches: feature/<task>
