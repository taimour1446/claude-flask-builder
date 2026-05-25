# Template — Dockerfile (multi-stage, non-root, gunicorn)

Path: `api/Dockerfile`. Pairs with `docker-entrypoint.sh.md`.

Covers rules: R110 (multi-stage, non-root, HEALTHCHECK, no `.git`/`.env`/
`tests` in image), R111 (entrypoint uses gunicorn, never `flask run`),
R112 (SIGTERM graceful shutdown).

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- builder stage ----------
FROM python:3.11-slim AS builder

# WHY virtualenv in /opt/venv: keeps a clean, copyable artifact for the
# runtime stage so the runtime image carries no build toolchain (R110).
ENV PIPENV_VENV_IN_PROJECT=0 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1

RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential gcc libpq-dev \
 && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /build
COPY Pipfile Pipfile.lock ./
RUN pip install --upgrade pip pipenv \
 && pipenv requirements > requirements.txt \
 && pip install -r requirements.txt

# ---------- runtime stage ----------
FROM python:3.11-slim AS runtime

# WHY tini: PID 1 reaper + signal forwarder so gunicorn workers get
# SIGTERM cleanly and the container shuts down gracefully (R112).
RUN apt-get update \
 && apt-get install -y --no-install-recommends libpq5 curl tini \
 && rm -rf /var/lib/apt/lists/*

# Non-root user (R110)
RUN groupadd --system --gid 1000 app \
 && useradd  --system --uid 1000 --gid app --home /app --shell /usr/sbin/nologin app

ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8000 \
    GUNICORN_WORKERS=4 \
    GUNICORN_TIMEOUT=30 \
    GUNICORN_GRACEFUL_TIMEOUT=20

COPY --from=builder /opt/venv /opt/venv

WORKDIR /app

# Copy ONLY what the runtime needs (R110: no .git/.env/tests in image —
# `.dockerignore` enforces this). Order from least to most likely to change
# for layer caching.
COPY --chown=app:app alembic.ini ./
COPY --chown=app:app migrations/   ./migrations/
COPY --chown=app:app seeds/        ./seeds/
COPY --chown=app:app app/          ./app/
COPY --chown=app:app docker-entrypoint.sh ./

RUN chmod +x docker-entrypoint.sh

USER app

EXPOSE 8000

# R110: HEALTHCHECK against the JSON health endpoint
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD curl -fsS "http://127.0.0.1:${PORT}/api/v1/health/check" || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "./docker-entrypoint.sh"]
```

## `.dockerignore` (ship alongside)

```gitignore
.git
.gitignore
.gitattributes
.github/
.env
.env.*
!.env.example
__pycache__/
*.pyc
.pytest_cache/
.coverage
htmlcov/
tests/
docs/
.vscode/
.idea/
.DS_Store
*.md
README.md
LICENSE
```

## Checklist (reviewer enforces)
- Multi-stage `FROM ... AS builder` + `FROM ... AS runtime` (R110).
- `USER app` non-root (R110).
- `HEALTHCHECK` present (R110).
- `ENTRYPOINT` runs `docker-entrypoint.sh` → gunicorn (R111).
- `tini` for signal forwarding (R112).
- `.dockerignore` excludes `.git`, `.env*` (except `.env.example`), `tests/`,
  `*.md` (R110).
- NEVER `flask run` anywhere (R111).
- `GUNICORN_GRACEFUL_TIMEOUT` env wired into `docker-entrypoint.sh` gunicorn
  invocation (R112).
