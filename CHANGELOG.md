# Changelog

## [1.0.8] — 2026-05-26

Round-8 audit: SHOWSTOPPER Constants fix + 3 NEW templates + Settings
hardening + 4 NEW reviewer checks.

- **SHOWSTOPPER fixed**: `Constants` were classes with `__str__` instance
  methods, but services raised `CustomValidationException(Constants.X)`
  passing the CLASS — `str(exc)` returned `"<class '...InvalidOTP'>"`
  instead of the message. **All auth error messages would have been
  broken.** Rewrote §14 as module-level UPPER_SNAKE_CASE string constants
  (`INVALID_TOKEN = "Invalid or expired token"`, etc.) — services raise
  `CustomValidationException(Constants.INVALID_TOKEN)` and the exception's
  default `__str__` returns the message string directly.
- **NEW templates**:
  - `templates/account-validation.py.md` — 6 auth-flow schemas (Signup/
    Login/ForgotPassword/ResetPassword/SendOTP/VerifyOTP) with centralized
    email lowercase + phone E.164 normalization, password length 12..72
    (matches bcrypt), OTP `^\d{6}$`, ptoken Length(43), enum-resistant
    signup uniqueness check.
  - `templates/conftest.py.md` — every fixture from `testing.md`
    implemented: app (session), db_session (SAVEPOINT rollback), client,
    test_user + test_admin (secrets-generated passwords), mock_stripe /
    mock_twilio / mock_fcm / mock_responses, mail_outbox. Critical:
    `os.environ.setdefault` runs BEFORE `create_application` import so
    `Settings.validate()` sees test env.
  - `templates/pyproject.toml.md` — `[tool.ruff].lint.select` matches the
    reviewer Stage 1 contract (E,F,I,B,N,UP,ANN,S,SIM,RUF), `[tool.black]`
    line-length=100, `[tool.pytest.ini_options]` testpaths + --cov-fail-
    under=60, `[tool.coverage.run] branch=true`.
- **Settings hardening** (`patterns.md §12 validate()`):
  - Rejects `"*"` in CORS_ORIGINS (env-driven R64 bypass closed).
  - Bounds `JWT_ACCESS_TTL_MINUTES` to 1..60 (R60 implicit cap).
  - Bounds `JWT_REFRESH_TTL_DAYS` to 1..90.
- **Logging hygiene** (`patterns.md §13`): `configure_logging` now
  `copy.deepcopy`s the module-level config dict before mutating
  `.root.level` — repeat calls (tests, reloads) no longer compound.
- **NEW reviewer checks** (`agents/flask-pattern-reviewer.md`):
  - **Stage 2 grep**: `Constants\.[A-Z][a-z]+[A-Z]\w*` (PascalCase
    Constants) → FAIL R15.
  - **Stage 2 grep**: in `app/domain/dto/AccountSchema.py` only,
    `_password|ptoken|otp(_|\b)|refresh_jti|device_token` → FAIL R67.
  - **Stage 3 READ**: `app/model/User.py` must have all 8 auth columns
    (_password, ptoken, ptoken_expires_at, otp, otp_created_at,
    otp_attempts, refresh_jti, deleted_at) + bcrypt `@hybrid_property
    password`.
  - **Stage 3 READ**: `Settings.validate()` must include the wildcard /
    JWT-bound / SECRET_KEY-length / ENV-set / CORS-non-empty gates.
- **Bug fixes**:
  - `auth-controller.py.md` companion service: 5 `Constants.PascalCase`
    refs converted to `Constants.UPPER_SNAKE_CASE`.
  - `validation-schema.py.md` + `patterns.md §4`: `Constants.NameTaken`
    → `Constants.NAME_TAKEN`.
  - `patterns.md §3` Service skeleton: `Role.Admin` → `Role.ADMIN`.
  - `user-model.py.md` password setter docstring documents bcrypt's
    72-byte truncation + recommends `validate.Length(min=12, max=72)`.
  - `model.py.md` "Auth-shaped lookup methods" subsection removed
    (duplicated `user-model.py.md`); replaced with a one-paragraph
    pointer to avoid drift.
  - `patterns.md §8`: explicit "Section-order note" explaining §10/§11
    physically precede §9 (intentional, do not renumber).

## [1.0.7] — 2026-05-26

Round-7 audit: NEW Settings/Logging/Constants skeletons, NEW User +
AccountSchema templates, 15 fixes.

- **NEW reference skeletons** (close 3 big "scaffolder must invent" gaps):
  - `patterns.md §12 Settings` — full `_Settings` class with `_csv` helper,
    `os.environ`-driven attrs, classmethod `validate()` (FAIL FAST on
    missing/short SECRET_KEY/invalid ENV/empty CORS_ORIGINS), singleton.
  - `patterns.md §13 Logging` — `_LOG_CONFIG_DEV` (plain) +
    `_LOG_CONFIG_PROD` (python-json-logger JsonFormatter) +
    `configure_logging(app, env, level)`.
  - `patterns.md §14 Constants` — `Role(str, Enum)` {ADMIN, CUSTOMER} +
    `_Msg` base + 11 concrete error-message classes consumed by
    `auth-controller.py.md` (InvalidToken / TokenExpired / InvalidOTP /
    OTPExpired / TooManyOTPAttempts / NameTaken / EmailTaken / PhoneTaken
    / InvalidCredentials / PermissionDenied / ReturnURLNotAllowed).
- **NEW templates** (close User-specific gaps):
  - `templates/user-model.py.md` — bcrypt hybrid password, ptoken / otp /
    refresh_jti / deleted_at columns, all four `get_by_*` static methods,
    explicit field assignment (R61).
  - `templates/account-schema.py.md` — whitelist-only DTO (id / email /
    phone / role / timestamps). Explicit "Forbidden fields" section
    rejects _password / ptoken* / otp* / refresh_jti / device_token.
- **Bug fixes (would have produced wrong code):**
  - `_REDACT_FIELDS` no longer lists `authorization` / `cookie` — they
    were dead code (interceptor never logs headers).
  - `model.py.md get_by_phone` docstring no longer claims E.164 normalization
    that doesn't happen — clarifies Validation layer is the normalizer (R26).
  - Reviewer R47 now has concrete grep heuristics for email/phone
    string-mangling soft-delete (was hand-wavey).
  - Reviewer R85 enumerates SDK-bearing providers
    {Stripe/Twilio/Firebase/AWS/GCS}; generic HTTP integrations are
    exempt (fixes false-positive on the integration-client template itself).
- **Agent updates aligning with Round-5 Configuration template:**
  - flask-feature-builder PLAN section 4: blueprint registration goes
    through `Configuration.register_blueprints` (not `application.py`).
  - flask-feature-builder Permitted-touch adds "User.get_by_* lookup when
    the feature requires it"; OFF-LIMITS rewritten with explicit carve-out.
  - flask-runner Step 6 accepts 200 OR 503 (degraded) — was only 200,
    would have false-FAILed health checks where a dep is intentionally
    down in tests.
  - flask-pattern-reviewer "Always read first" now matches templates to
    the change layer (auth → auth-controller, deploy → dockerfile, etc.).
  - flask-scaffolder Procedure cross-refs `patterns.md §1/§7/§9/§10/§11/
    §12/§13/§14` + Required Barrels subsection calling out
    `scheduler_jobs/__init__.py` (without which `@scheduler.task`
    decorators never run).
- **scaffold-checklist:** Settings step gains explicit `_csv()` helper
  code + cross-ref to `patterns.md §12`.
- **.gitignore:** adds `.pytest_cache/`, `.coverage`, `.coverage.*`,
  `htmlcov/`, `.ruff_cache/`, `.mypy_cache/`.
- **SKILL.md** templates list adds `user-model.py.md` + `account-schema.py.md`.

## [1.0.6] — 2026-05-26

Round-6 audit: SQLAlchemy 2.0 fixes, Dockerfile runtime bug, 9 reviewer
orphans closed, PII redaction, Stripe webhook bytes fix.

- **Bug fixes (would have broken real deploys):**
  - `docker-entrypoint.sh` no longer uses `pipenv run` — pipenv is NOT
    installed in the runtime stage; the venv is on PATH. Both `flask db
    upgrade` and `gunicorn` are called directly.
  - `db.text(...)` → `from sqlalchemy import text` + `text(...)` in
    `distributed_lock` (patterns.md §9) and `configuration.py.md` health
    endpoint (SQLAlchemy 2.0 removed the passthrough).
  - `SESSION_COOKIE_SECURE` is now conditional on `settings.ENV ==
    "production"` (was unconditional True; broke dev sessions over HTTP).
- **Reviewer coverage:** Stage 3 now has explicit READs for 11 previously-
  orphan rules: R40 (WHERE column indexed), R47 (deleted_at not
  email-mangling), R48 (synchronize_session + expire_all), R63 (file
  upload allowlist + size + image-bomb), R65 (password reset strength +
  TTL + single-use), R66 (OTP TTL + max-attempts + constant-time
  compare), R72 (admin role assertion), R82 (response.json wrapped),
  R84/R85 (Session singleton + vendor SDK), R90 (errorhandler types),
  R92 (health JSON shape).
- **Security hardening:**
  - `_REDACT_FIELDS` adds PII (email, phone, ssn, credit_card,
    national_id, dob, authorization, cookie).
  - `RequestInterceptor` now redacts `request.args` (query strings) in
    addition to JSON body — secrets in URLs no longer leak to access logs.
  - `webhook.py.md` Stripe handler uses `request.get_data(as_text=False)`
    — Stripe HMAC is computed over raw bytes; decoding could break
    signature verification on edge cases.
  - `integration-client.py.md` documents that `Idempotency-Key` is
    provider-specific (Stripe canonical; others vary).
- **Foundation completeness:**
  - `model.py.md` gains `get_by_email`, `get_by_phone`, `get_by_ptoken`,
    `get_by_refresh_jti` — required by `auth-controller.py.md`'s
    companion service.
  - `scaffold-checklist.md` step 3 gains comprehensive Settings attr
    table (APP_NAME/APP_VERSION/ENV/LOG_LEVEL/SECRET_KEY/DATABASE_URI/
    JWT_*/CORS_ORIGINS/MAX_CONTENT_LENGTH_BYTES/MAIL_*/STRIPE_*) +
    Settings.validate() contract.
  - `scaffold-checklist.md` step 2 gains concrete barrel skeletons for
    every layer (api, service, validation, dto, model, scheduler_jobs).
    `scheduler_jobs/__init__.py` MUST import each job to register
    `@scheduler.task` decorators.
  - `scaffold-checklist.md` step 5 documents `refresh_jti` size choice
    (`secrets.token_urlsafe(16)` ~22 chars fits String(32) with headroom).
  - `scaffold-checklist.md` step 6 lists all 8 auth endpoints matching
    `auth-controller.py.md`.
- **Doc consistency:**
  - `patterns.md §7` prose now says `Auth.generate_tokens(user)` (plural).
  - `patterns.md` gains `## 8. (reserved)` marker between §7 and §10 so
    cross-refs to §1/§7/§9/§10/§11 remain stable.
  - `testing.md` clarifies `api/` is the pytest cwd; `app/tests/` is the
    testpaths relative to that.

## [1.0.5] — 2026-05-25

Round-5 audit: convergence + new templates.

- NEW templates: `dockerfile.md`, `docker-entrypoint.sh.md`,
  `auth-controller.py.md`, `configuration.py.md` — covers R60/R63/R65/R66/
  R74/R90/R92/R110/R111/R112 that previously had no template.
- Reviewer Stage 2: added R21 (pickle.loads), R34 (TODO/FIXME), and
  literal-secret patterns (sk_live_/pk_live_/sk_test_/whsec_/AKIA*/ghp_).
- Reviewer Stage 3: added R60 (JWT exp claim), R67 (RequestInterceptor
  redaction usage), R69 (Stripe return_url allowlist), R83 (no hardcoded
  external URLs).
- Reviewer Stage 2 now explicitly requires `grep -E` (ERE).
- `flask-runner` Step 5a excludes both HEAD AND OPTIONS to avoid false-FAIL R100.
- `patterns.md`: consolidated duplicate §8; added `distributed_lock`
  skeleton (Postgres advisory lock); fixed `_REDACT_FIELDS` case-insensitive
  match; fixed RequestInterceptor `except` to specific exceptions (no more
  bare `except Exception`).
- `scaffold-checklist.md`: User model gains `refresh_jti` column required by
  `Auth.generate_tokens`. Settings docs the `app.config["ENV"]` mirror.
- `conventions.md`: STRIPE_SECRET → STRIPE_SECRET_KEY (leftover from Round-4).
- R45 marked as deliberately redundant with R19.
- `env-example.md` SECRET_KEY comment now actionable.

## [1.0.4] — 2026-05-25

Round-4 audit: reviewer enforceability sweep + foundation utils.

- Reviewer Stage 1: ruff ruleset spelled out (E,F,I,B,N,UP,ANN,S).
- Reviewer Stage 2: +12 grep checks (R22 shell=True, R44 default=callable(),
  R46 DateTime tz, R50 migration slug, R64 CORS, R70 webhook signature, R71
  S3 public-read, R73 |safe, R74 cookies (Stage 3), R110 Dockerfile hardening,
  R112 SIGTERM, plus migration filename slug enforcement).
- Reviewer NEW "PRE-CHECK-only enforcement" block honestly scopes R10–R16,
  R23–R28, R30–R34, R140–R142 to plan-review (no over-claim).
- `flask-runner`: NEW Steps 5a (R100 endpoint enumeration) + 5b
  (R101 Validation cross-ref).
- `flask-feature-builder`: 8-section PLAN format with Permitted-touch /
  OFF-LIMITS lists; explicit STOP marker; FAIL-by-stage guidance.
- `patterns.md`: full skeletons for BaseResponse, Auth/token_required,
  Validation/CustomValidationException, RequestInterceptor with redaction
  allowlist.
- `data-patterns.md`: composite-UNIQUE migration syntax + docstring waiver.
- `testing.md`: split scaffold-time vs reference endpoints; added pyproject
  `[tool.pytest.ini_options]` block.
- `versions.md`: urllib3 ≥1.26 pin.
- Templates: return type hints on model/seeder/validation; STRIPE_SECRET_KEY +
  STRIPE_PUBLISHABLE_KEY standardized; urllib3 version contract comment.

## [1.0.3] — 2026-05-24

Round-3 audit: enforceability + R43 clarity + integration template.

- NEW `templates/integration-client.py.md` (generic HTTP client + Stripe
  mutation snippet). Covers R17/R18/R20/R80/R82/R83/R84/R85/R81/R69.
- Reviewer PRE-FAILs missing barrel/blueprint/migration registration.
- Reviewer R100/R101/R102 moved to runner scope; R05/R41/R91 marked ADVISORY.
- `coding-standards` R43 rewritten: distinguishes repository-style
  (@staticmethod) from row-instance (self) methods.
- `flask-scaffolder` uses skill-relative paths; returns control to
  orchestrator; never self-audits.
- `flask-runner` description aligned to `flask db upgrade`.

## [1.0.2] — 2026-05-24

Round-2 audit: 9 self-audit gaps fixed.

- Return type hints across 4 templates.
- Reviewer rewritten with 3-stage approach (ruff/black → grep → semantic READ).
- SKILL.md acknowledges Foundation utils synthesized from
  scaffold-checklist + patterns + security.
- Added CONTRIBUTING.md, CHANGELOG.md, .editorconfig.

## [1.0.0] — 2026-05-20

Initial release.

- 1 skill (SKILL.md + 10 reference files + 12 templates)
- 4 agents: flask-scaffolder, flask-feature-builder, flask-pattern-reviewer, flask-runner
- Locked stack: Python 3.11+, Flask, pipenv, SQLAlchemy, Alembic, Marshmallow, JWT, pytest, gunicorn
- Encoded ~334 patterns from a 25-pass audit of a real production Flask codebase
- Double-gated review loop (PRE-CHECK plan + POST-CHECK code + runner verify)
- Self-marketplace published at taimour1446/claude-flask-builder
