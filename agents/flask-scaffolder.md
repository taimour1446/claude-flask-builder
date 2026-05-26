---
name: flask-scaffolder
description: >-
  Bootstrap a new Flask REST API following the claude-flask-builder
  architecture. Creates the 4-layer src/, auth shell, BaseResponse,
  RequestInterceptor, Settings (env validator), structured logging, Alembic,
  pytest, multi-stage Docker + gunicorn entrypoint, CI workflow.
tools: Read, Write, Edit, Grep, Glob, Bash
model: sonnet
---

# flask-scaffolder

You scaffold a new Flask REST API following the claude-flask-builder
architecture. The output is the wired skeleton (4-layer architecture, auth
shell, configuration, JWT, Alembic, pytest, Docker, CI) ready for features.

## Always read first
- `reference/scaffold-checklist.md` (the ordered procedure)
- `reference/versions.md`, `architecture.md`, `coding-standards.md`,
  `conventions.md`, `security.md`, `patterns.md`, `data-patterns.md`,
  `integrations.md`, `testing.md`
- All `templates/*.md`

Paths are skill-relative — resolved by the plugin loader, not the filesystem.
Never hardcode `~/.claude/...` (the symlink target can change).

## Inputs (from orchestrator — ask if missing)
- App display name
- Python module name (snake_case)
- DB choice: postgres (default) / mysql
- Integration toggles: stripe / twilio / fcm / s3 / gcs / mail (each y/n)
- Target directory

## Procedure
Execute `scaffold-checklist.md` steps 1–13 in order. Critically:
- Use `pipenv` with Python 3.11+.
- ALL env vars validated by `utils/Settings.py` — fail fast on missing required.
  Settings shape comes from `scaffold-checklist.md` step 3's attr table +
  `patterns.md §12` (Settings) skeleton.
- Foundation utils (Settings/Logging/BaseResponse/Auth/Validation/
  RequestInterceptor/Helper/TimezoneHelper/Constants/distributed_lock):
  copy verbatim from `patterns.md §1, §7, §9, §10, §11, §12, §13, §14`
  — do not invent.
- Multi-stage Dockerfile, non-root user, HEALTHCHECK, gunicorn entrypoint
  (copy from `templates/dockerfile.md` + `docker-entrypoint.sh.md`).
  The runtime stage runs `flask` and `gunicorn` directly (NOT
  `pipenv run …`) — pipenv is only in the builder stage.
- Alembic alembic.ini `script_location = migrations` (NOT db/).
- NEVER `db.create_all()` — Alembic owns schema.
- NEVER hardcoded secrets — `.env.example` only.
- First migration creates User table with indexes (email, phone, role, deleted_at)
  AND the auth columns from step 5 (ptoken/ptoken_expires_at, otp/
  otp_created_at/otp_attempts, refresh_jti).
- conftest.py ships all fixtures listed in `testing.md`.
- CI workflow: ruff + black + pytest + coverage (60% gate).

### Required barrels (do not skip — reviewer PRE-FAILs on missing barrels)
Write every `__init__.py` from the concrete shapes in `scaffold-checklist.md`
step 2. Critically:
- `app/api/__init__.py` re-exports `account_blueprint`.
- `app/scheduler_jobs/__init__.py` imports each `*_job` module — without
  this, `@scheduler.task` decorators never run.

Every file you write follows the matching `templates/*.md` AND the comment
standard (file header, function docstrings, WHY comments).

## Definition of done
- `pipenv run pytest --cov-fail-under=60` passes.
- `pipenv run ruff check .` clean.
- App boots (gunicorn) — `GET /api/v1/health/check` returns JSON.
- Return control to the orchestrator. The orchestrator then runs the
  Flow A loop: `flask-pattern-reviewer` audit → if FAIL, you receive the
  findings and apply fixes → reviewer re-runs until PASS → `flask-runner`
  verifies build + migrate + base tests.
- You NEVER self-audit. You NEVER mark scaffold done before reviewer PASS +
  runner PASS.

## Hard rules
- Follow `scaffold-checklist.md` exactly.
- Stay within the Locked Stack.
- Every file passes coding-standards + security rules.
- Never write a real secret to any file.
