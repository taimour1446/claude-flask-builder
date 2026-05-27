# Versions — known-good baseline

| Package | Version | Notes |
| --- | --- | --- |
| Python | 3.11+ | 3.9 EOL |
| Flask | 3.0.x | (reference had 2.2 — outdated) |
| SQLAlchemy | 2.0.x | pinned (reference had * — bad) |
| Flask-SQLAlchemy | 3.1.x | |
| Flask-Migrate | 4.0.x | |
| Alembic | 1.13.x | |
| Flask-Seeder | latest | |
| Marshmallow | 3.20.x | |
| flask-smorest | 0.42.x | OpenAPI 3.x + Swagger UI; reads `@blp.arguments(<Validation>)` + `@blp.response(<DTO>)` to AUTO-GENERATE the spec from existing Marshmallow schemas — no hand-written YAML |
| apispec | 6.x | flask-smorest dependency; pinned because flask-smorest API surface assumes 6.x |
| apispec-webframeworks | latest | flask-smorest dependency |
| Flask-Mail | 0.10.x | |
| Flask-CORS | 4.0.x | |
| Flask-APScheduler | latest | |
| pyjwt | 2.8+ | |
| bcrypt | 4.x | |
| gunicorn | 21.x | mandatory prod server |
| stripe | latest stable | |
| twilio | latest stable (SDK, not hand-rolled) |
| firebase-admin | latest | for FCM |
| boto3 | latest | |
| google-cloud-storage | latest | |
| pytest, pytest-flask, pytest-cov | latest | |
| ruff, black, isort | latest | |
| urllib3 | >=1.26, <3 | vendored by requests; `Retry(allowed_methods=...)` API used in `templates/integration-client.py.md`. <1.26 used `method_whitelist=` — incompatible |
| python-json-logger | latest | structured logs |
| sentry-sdk[flask] | latest | optional |
| pip-audit | latest | CI dep scan |

Removed vs reference: pip, install (garbage placeholders), colorama, termcolor, importlib-metadata, dataclasses-json, mysql-connector-python (only if mysql chosen).
