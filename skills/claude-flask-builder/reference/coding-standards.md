# Coding Standards — the strict ruleset

Reviewer enforces ZERO tolerance. Each rule has an id `[Rxx]` for findings.
Every rule BLOCKING unless marked `[ADVISORY]`.

## 1. Formatting & imports
- **R01** 4-space indent; no tabs.
- **R02** Black + Ruff format (line length 100).
- **R03** Imports ordered: stdlib → third-party → app (3 groups, blank-line separated).
- **R04** `from X import *` is FORBIDDEN.
- **R05** Type hints on every public function signature.

## 2. Naming
- **R10** Modules `snake_case.py`; classes `PascalCase`; functions/vars `snake_case`.
- **R11** Controllers end in `Controller` (file `XController.py`, blueprint factory `x_blueprint`).
- **R12** Services end in `Service` (class `XService`, file `XService.py`).
- **R13** Models singular PascalCase (`User`, `Order`); tablename plural snake_case (`users`, `orders`). NO exceptions (Business was singular in reference — wrong).
- **R14** Schemas end in `Schema`. Validations end in `Validation`.
- **R15** Constants UPPER_SNAKE_CASE in `utils/constants.py`.
- **R16** Booleans read as predicates: `is_active`, `has_token`.

## 3. Python & TypeScript discipline
- **R17** No `bare except:` — catch specific exceptions (`SQLAlchemyError`, `requests.RequestException`, `stripe.error.StripeError`, etc.).
- **R18** No `print()` in src — use `logging` with named logger.
- **R19** Money columns: `Numeric(10,2)` or `Decimal`. NEVER `Float`.
- **R20** Random for tokens/passwords: `secrets` module. NEVER `random.randrange`.
- **R21** No `pickle.loads` on untrusted data.
- **R22** No `shell=True` in subprocess.

## 4. Layer boundaries (the 4-layer rule)
- **R23** Controllers contain ZERO business logic. Try/except → service call → BaseResponse.
- **R24** Services contain ZERO HTTP/Flask code; take parsed dicts, return parsed dicts.
- **R25** Models own their own DB transactions (single-row); services own multi-model transactions (single commit at end).
- **R26** Validation = Marshmallow request schemas. Serialization = Marshmallow DTO schemas. NO mixing.
- **R27** `@post_dump` may NEVER hit the DB. Pre-join in the service.
- **R28** `@validates_schema` raises `CustomValidationException` (not `ValidationError`).

## 5. MANDATORY COMMENTING
- **R30** Every file: header docstring.
- **R31** Every function/class: docstring with intent + non-obvious WHY.
- **R32** Every complex/non-obvious block: inline `#` comment.
- **R33** No commented-out code.
- **R34** No `TODO`/`FIXME` unresolved in delivered code.

## 6. Database
- **R40** Every column used in WHERE has an index in the migration that adds it.
- **R41** Composite UNIQUE constraint on every "must not duplicate" pair (e.g. `(order_id, driver_id)` on OrderRequest, `(user_id, slot_id, date)` on DriverSlot, `(key)` on Fare).
- **R42** Every text() uses bind params (`:name`) — NO `.format`/`f-string`/`%`.
- **R43** Every model method = `@staticmethod`. NO exceptions.
- **R44** `default=callable` for runtime defaults (NOT `default=Helper.generatePin()` which evaluates at class-load time).
- **R45** Money columns `Numeric(10,2)`.
- **R46** Timestamps tz-aware (`DateTime(timezone=True)`).
- **R47** Soft-delete pattern: explicit `deleted_at` column, NOT string-mangling on email/phone.
- **R48** Bulk `.update(... synchronize_session=False)` is followed by `db.session.expire_all()` or refetch.
- **R49** Alembic downgrade() is the symmetric reverse of upgrade() — never `pass`.
- **R50** Migration files use descriptive slugs (`flask db migrate -m "..."`).

## 7. Security
- **R60** JWT includes `exp` claim (TTL 15-60 min); refresh-token rotation enforced.
- **R61** No mass-assignment: `Model(field1=data['field1'], ...)` only — NEVER `model.__dict__.update(data)`.
- **R62** Marshmallow `unknown=EXCLUDE` (not INCLUDE) for request validation.
- **R63** File upload: MIME-type allowlist + max size + image-bomb protection (`Image.MAX_IMAGE_PIXELS`).
- **R64** CORS: explicit `origins=[...]` from env, never wildcard with credentials.
- **R65** Password reset token: ≥32 bytes (`secrets.token_urlsafe(32)`), TTL ≤15 min, single-use.
- **R66** OTP: TTL ≤5 min, max-attempts rate limit.
- **R67** RequestInterceptor redacts password/card/token/otp fields before logging.
- **R68** No `traceback.format_exc()` in client response — dev-only; prod returns ticket id.
- **R69** Every external-URL Stripe return validated against an env allowlist (open-redirect).
- **R70** Stripe webhook signature verified with `stripe.Webhook.construct_event`.
- **R71** S3 uploads default `ACL='private'` + presigned URLs. `public-read` only when explicit.
- **R72** Every admin route asserts role explicitly in the service.
- **R73** No `|safe` filter on user-controlled content in Jinja — use bleach.
- **R74** Flask session: `SESSION_COOKIE_SECURE`, `HTTPONLY`, `SAMESITE='Lax'`.

## 8. External APIs (resilience)
- **R80** Every `requests.*` call: `timeout=N` explicit (5-30s).
- **R81** Every Stripe mutation: `idempotency_key=` set.
- **R82** Every `response.json()`: wrapped in try/except for `JSONDecodeError`.
- **R83** No hardcoded URLs for external services — all env-driven.
- **R84** `requests.Session()` per-provider singleton (connection reuse).
- **R85** Use vendor SDK (twilio package) — no hand-rolled HTTP.

## 9. Errors & observability
- **R90** Global `@errorhandler` registered for: `SQLAlchemyError`, `IntegrityError`, `ValidationError`, `CustomValidationException`, `jwt.ExpiredSignatureError`, `jwt.InvalidTokenError`, generic `Exception`.
- **R91** Every response carries `X-Request-ID` header.
- **R92** Health endpoint returns JSON `{ ok, version, deps }` — never plain string.
- **R93** Optional Sentry hook in `Configuration.py` (env-gated).

## 10. Tests
- **R100** Every new endpoint: at least one pytest E2E in `tests/`.
- **R101** Every new Validation class: at least one error-message assertion test.
- **R102** Coverage ≥60% to pass CI.

## 11. Deploy
- **R110** Dockerfile: non-root user, HEALTHCHECK, multi-stage, no .git/.env/tests in image.
- **R111** Entrypoint: wait-for-db → alembic upgrade → gunicorn (NEVER `flask run`).
- **R112** SIGTERM graceful shutdown configured.

## 12. Scope discipline
- **R140** Surgical changes only. No unrelated refactors.
- **R141** No new dependency outside the Locked Stack without explicit user approval.
- **R142** No files outside the architecture layout.

## Reviewer verdict
- **PASS** — "PASS — all standards satisfied."
- **FAIL** — one line per finding: `FAIL [Rxx] <file>:<line> — <wrong> → <fix>`.
