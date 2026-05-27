# claude-flask-builder

> 🤖 A **Claude Code plugin** — an AI agent system that scaffolds and extends
> production-grade **Flask REST APIs** on a fixed, opinionated 4-layer
> architecture (Controller / Service / Validation+DTO / Model).

It ships **one skill** and **four specialized AI agents** that work under a
self-reviewing build loop. The skill encodes ~330 specific corrections
distilled from 26 audit passes of a real production Flask codebase — so every
scaffolded app starts with the security, correctness, and operational gaps
already fixed.

## What it does

- **Scaffolds** a new Flask app: app factory, 4-layer architecture, JWT auth,
  Alembic, Flask-Seeder, RequestInterceptor, BaseResponse, Marshmallow
  validation+DTO, Stripe/Twilio/S3/GCS/Mail integration stubs, Docker,
  gunicorn entrypoint, pytest, CI.
- **Extends** an existing app: adds a Controller + Service + Validation +
  DTO + Model + Alembic migration + pytest E2E, all gated by the reviewer.
- **Reviews** code against a numbered ruleset (R01–R150+).
- **Builds + tests** with `flask-runner`: pipenv install → DB up → Alembic
  upgrade → pytest with coverage, reports pass/fail.

## The locked stack

| Concern | Choice |
| --- | --- |
| Language | Python 3.11+ |
| Framework | Flask (app factory + blueprints) |
| Package manager | pipenv |
| ORM | SQLAlchemy via Flask-SQLAlchemy |
| Migrations | Alembic via Flask-Migrate |
| Seeders | Flask-Seeder (idempotent) |
| DB | PostgreSQL default, MySQL switchable |
| Validation | Marshmallow (request schemas with error_messages) |
| Serialization | Marshmallow (DTO schemas with @post_dump — NO DB queries inside) |
| Auth | JWT (pyjwt) with exp + refresh tokens |
| Money | Numeric / Decimal (NOT Float) |
| Response | BaseResponse envelope (200 + error flag) — kept for client compat |
| Logging | Structured stdlib logging (NO print) |
| Container | Multi-stage Dockerfile, non-root user, HEALTHCHECK, gunicorn |
| Tests | pytest + coverage (60% gate to start) |

## The agents

| Agent | Role |
| --- | --- |
| `flask-scaffolder` | Bootstraps a new Flask app |
| `flask-feature-builder` | Builds a vertical resource — gated by reviewer on both ends |
| `flask-pattern-reviewer` | Strict standards gate (read-only) |
| `flask-runner` | Builds, migrates, runs pytest, reports pass/fail |

## The double review gate

`flask-feature-builder` cannot write code until `flask-pattern-reviewer`
has approved its **plan**, and cannot finish until the reviewer has approved
its **code**. Failures loop back until they pass.

## Auto-generated API docs (Swagger UI)

Every scaffolded app ships **Swagger UI at `/api/v1/docs`** and the raw
OpenAPI 3 spec at `/api/v1/openapi.json` — both by default, no opt-in.

The spec is **auto-generated from the Marshmallow schemas** the skill
already requires for request validation (R26) and response serialization
(R27). When `flask-feature-builder` adds a new endpoint, the
`@blp.arguments(<ValidationSchema>)` + `@blp.response(<status>, <DTO>)`
decorators on the controller method tell `flask-smorest` to reflect those
exact schemas into the OpenAPI spec — no hand-written YAML, no
duplication, no drift between validation and docs.

**Persistent JWT auth in Swagger UI**: after a single `/api/v1/auth/login`
call, paste the access token into Swagger UI's Authorize dialog once.
`persistAuthorization=true` saves it to browser `localStorage`, and every
subsequent "Try it out" request auto-attaches `Authorization: Bearer
<token>` — no re-pasting per request, even across browser restarts.

In production, the UI is gated behind either an env-set
`OPENAPI_DOCS_TOKEN` query param or any valid Bearer login; the
`openapi.json` endpoint stays publicly available for CI / SDK
generators.

## Installation

```
/plugin marketplace add taimour1446/claude-flask-builder
/plugin install claude-flask-builder@taimour1446
```

Once installed, trigger with `/claude-flask-builder`, or just ask Claude to
build or extend a Flask app.

## See also

- [CHANGELOG.md](CHANGELOG.md) — version history; the current line is documented per round.
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to propose changes; the
  audit-driven workflow (`/audit-loop` + double-gated review) is the
  source of truth for what lands.

## License

MIT — see [LICENSE](LICENSE).
