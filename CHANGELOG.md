# Changelog

## [1.0.0] — 2026-05-20

Initial release.

- 1 skill (SKILL.md + 10 reference files + 12 templates)
- 4 agents: flask-scaffolder, flask-feature-builder, flask-pattern-reviewer, flask-runner
- Locked stack: Python 3.11+, Flask, pipenv, SQLAlchemy, Alembic, Marshmallow, JWT, pytest, gunicorn
- Encoded ~334 patterns from a 25-pass audit of a real production Flask codebase
- Double-gated review loop (PRE-CHECK plan + POST-CHECK code + runner verify)
- Self-marketplace published at taimour1446/claude-flask-builder
