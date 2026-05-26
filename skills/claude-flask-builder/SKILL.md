---
name: claude-flask-builder
description: >-
  Build and extend production-grade Flask REST APIs on a fixed 4-layer
  architecture (Controller / Service / Validation+DTO / Model). Use whenever
  the user asks to create, scaffold, extend, review, or deploy a Flask app, or
  add an endpoint / model / service / migration / pytest. Enforces 330+
  corrections distilled from a real production codebase via a double-gated
  review loop.
---

# claude-flask-builder

This skill builds and extends production-grade Flask REST APIs that follow ONE
fixed architecture, ONE strict coding standard, and ~330 corrections derived
from a 26-pass audit of a real production Flask codebase. The skill never
asks the user to pick libraries — the stack is locked.

The skill is self-contained — every rule lives in this skill's files.

## When this skill applies

- Scaffold a new Flask REST API.
- Extend an existing app of this architecture — add a Controller, Service,
  Validation, DTO, Model, Migration, Seeder, pytest.
- Review Flask code against the standard.
- Build, migrate, run pytest, deploy.

## Locked stack

| Concern | Choice |
| --- | --- |
| Language | Python 3.11+ |
| Framework | Flask (app factory + blueprints) |
| Package manager | pipenv |
| ORM | SQLAlchemy via Flask-SQLAlchemy |
| Migrations | Alembic via Flask-Migrate |
| Seeders | Flask-Seeder (idempotent) |
| DB | PostgreSQL default, MySQL switchable at scaffold |
| Validation | Marshmallow request schemas (error_messages set) |
| Serialization | Marshmallow DTO (`@post_dump` may NOT query DB) |
| Auth | JWT (pyjwt) with `exp` claim + refresh token rotation |
| Money | Numeric / Decimal (NEVER Float) |
| Response | BaseResponse envelope (200 + error flag) — locked for client compat |
| Logging | Structured stdlib `logging` (NEVER `print()`) |
| Cron | APScheduler + distributed lock for multi-pod |
| Container | Multi-stage Dockerfile, non-root, HEALTHCHECK, gunicorn |
| Tests | pytest + coverage (60% gate) |

## Reference files — read before acting

| File | Read when |
| --- | --- |
| `reference/coding-standards.md` | ANY code task — strict ruleset, reviewer's rubric |
| `reference/architecture.md` | Folder layout, layer boundaries |
| `reference/conventions.md` | Naming, barrels, env, Docker, CI, git |
| `reference/patterns.md` | BaseResponse, token_required, RequestInterceptor, Service shape |
| `reference/data-patterns.md` | SQLAlchemy patterns, transactions, indexes, migrations |
| `reference/integrations.md` | Stripe / Twilio / FCM / S3 / GCS / Google Maps / TollGuru |
| `reference/security.md` | The 50+ security corrections vs the reference. Read for ANY code. |
| `reference/testing.md` | pytest fixtures, the 10 must-cover endpoints, mocking strategy |
| `reference/scaffold-checklist.md` | Ordered steps for a new app |
| `reference/versions.md` | Pinned version baseline |

`coding-standards.md`, `conventions.md`, `security.md` are mandatory for every
code task.

## Templates

`templates/controller.py.md`, `service.py.md`, `validation-schema.py.md`,
`dto-schema.py.md`, `model.py.md`, `user-model.py.md`,
`account-schema.py.md`, `migration.py.md`, `pytest-endpoint.py.md`,
`seeder.py.md`, `webhook.py.md`, `cron-job.py.md`, `app-factory.py.md`,
`configuration.py.md`, `auth-controller.py.md`,
`integration-client.py.md`, `dockerfile.md`, `docker-entrypoint.sh.md`,
`env-example.md`.

`user-model.py.md` is the **specific** User model shape (extends the
generic `model.py.md`). It includes the bcrypt-hybrid password, all auth
columns (ptoken/otp/refresh_jti/deleted_at), and the four `get_by_*`
lookup methods consumed by `auth-controller.py.md`.

`account-schema.py.md` is the only DTO permitted to serialize a User row.
It is a strict whitelist (id/email/phone/role/timestamps) — the reviewer
FAILs any addition of `_password`, `ptoken*`, `otp*`, `refresh_jti`, or
`device_token`.

`integration-client.py.md` is the mandatory shape for any new external-API
wrapper in `app/integrations/` — it embodies R17/R18/R20/R80/R82/R83/R84/R85
(and R81/R69 for Stripe). Reviewer FAILs any `requests.*` call that lives
outside an integration client file unless explicitly justified.

`auth-controller.py.md` is the canonical shape for `AccountController` and
covers R60/R65/R66 (JWT exp, password-reset token strength + TTL + single-use,
OTP TTL + max-attempts + constant-time compare).

`configuration.py.md` realizes R74 (session cookie flags), R90 (global
errorhandlers), R92 (health JSON), and the `app.config["ENV"]` plumbing
that `BaseResponse.respondError` reads for the prod trace gate.

`dockerfile.md` + `docker-entrypoint.sh.md` realize R110 (multi-stage,
non-root, HEALTHCHECK), R111 (gunicorn — never `flask run`), R112 (SIGTERM
graceful shutdown via `--graceful-timeout` + tini).

**Foundation utils** referenced in `scaffold-checklist.md` step 3 (Settings,
Logging, BaseResponse, Auth, Validation, RequestInterceptor, Helper,
TimezoneHelper, Constants, Configuration.py) are not separate template files —
the scaffold-checklist itself + the patterns in `patterns.md` + `security.md`
fully specify their shape. `flask-scaffolder` synthesizes them from those
sources. The patterns are exhaustive enough that an additional skeleton per
file would be redundant.

## Agents

| Agent | Role |
| --- | --- |
| `flask-scaffolder` | Bootstrap a new Flask app |
| `flask-feature-builder` | Build a vertical resource — gated by reviewer both ends |
| `flask-pattern-reviewer` | Strict standards gate, read-only |
| `flask-runner` | Build + migrate + pytest + report |

## Orchestration

### Flow A — Scaffold a new app
1. Ask user for: app name, Python module name, DB choice (postgres/mysql),
   target dir, Stripe/Twilio/FCM/S3/GCS toggles.
2. Read `scaffold-checklist.md`, `versions.md`, `security.md`.
3. Delegate to `flask-scaffolder`.
4. Hand to `flask-pattern-reviewer`. Loop until PASS.
5. Hand to `flask-runner` to confirm app builds + migrates + base tests pass.

### Flow B — Build or change a feature (double gate)
1. Read `coding-standards.md`, `architecture.md`, `patterns.md`,
   `data-patterns.md`, `security.md`, and the relevant `templates/*.md`.
2. `flask-feature-builder` produces a PLAN — files + key snippets only.
3. PRE-CHECK: `flask-pattern-reviewer` audits the plan. Loop until PASS.
4. `flask-feature-builder` writes code + Alembic migration + pytest E2E.
5. POST-CHECK: `flask-pattern-reviewer` audits the code. Loop until PASS.
6. RUN: `flask-runner` runs the app + pytest. If FAIL → back to step 4.
7. Done only when both reviews PASS and runner reports PASS.

### Flow C — Review existing code
Delegate to `flask-pattern-reviewer`. Report findings.

### Flow D — Build / test / deploy
Delegate to `flask-runner`.

## Hard rules (the reviewer enforces all)

1. Follow `coding-standards.md` exactly (R01–R150+).
2. Mandatory commenting: file header, docstring on every function/class,
   WHY comments on every non-obvious block.
3. NO `print()` — structured logger only.
4. NO `Float` for money — `Numeric(10,2)` / `Decimal`.
5. NO `text("...{val}...".format(...))` — bind params only.
6. NO mass-assignment via `model.__dict__.update(data)` — explicit fields.
7. NO `unknown=INCLUDE` propagating into model construction.
8. NO `bare except:` — catch specific exceptions.
9. NO `random` for security tokens — `secrets` module.
10. NO `print()` ever. NO `Float` for money. NO secrets in code.
11. Every CREATE/DELETE mutation uses an idempotency key (Stripe) or DB unique constraint.
12. Every external API call: explicit timeout + retry + circuit-breaker.
13. Every list endpoint: pagination + index on filter columns.
14. Every Alembic migration: working downgrade() (never `pass`).
15. Every Docker image: non-root user + HEALTHCHECK + gunicorn (not flask run).
16. Locked stack only — no new dependency without explicit user approval.
17. Surgical changes only — no unrelated refactors.
