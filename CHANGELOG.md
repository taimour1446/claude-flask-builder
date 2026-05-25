# Changelog

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
