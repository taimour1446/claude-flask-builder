# Template — docker-entrypoint.sh

Path: `api/docker-entrypoint.sh`. Invoked by `Dockerfile` ENTRYPOINT via tini.

Covers R111 (gunicorn, never `flask run`), R112 (SIGTERM graceful shutdown
via `--graceful-timeout`), and database-readiness gating.

```bash
#!/usr/bin/env bash
# docker-entrypoint.sh — wait-for-db → alembic upgrade → exec gunicorn

# WHY -E: propagate ERR trap into functions/subshells.
# WHY -u: undefined-variable usage is a deploy bug, fail fast.
# WHY -o pipefail: a failing alembic in a pipeline must abort the boot.
set -Eeuo pipefail

: "${DATABASE_URI:?DATABASE_URI is required}"
: "${PORT:=8000}"
: "${GUNICORN_WORKERS:=4}"
: "${GUNICORN_TIMEOUT:=30}"
: "${GUNICORN_GRACEFUL_TIMEOUT:=20}"
: "${FLASK_APP:=app.application:create_application}"

log() { printf '[entrypoint] %s\n' "$*"; }

# ---- 1. wait-for-db ----
# WHY a tiny Python check (not pg_isready): works for postgres AND mysql
# without adding either client binary to the image.
log "waiting for database…"
python - <<'PY'
import os, sys, time
from urllib.parse import urlparse
import socket

uri = urlparse(os.environ["DATABASE_URI"])
host = uri.hostname or "localhost"
port = uri.port or (5432 if uri.scheme.startswith("postgres") else 3306)
deadline = time.monotonic() + 60
while time.monotonic() < deadline:
    try:
        with socket.create_connection((host, port), timeout=2):
            print(f"[entrypoint] db reachable at {host}:{port}")
            sys.exit(0)
    except OSError:
        time.sleep(1)
print(f"[entrypoint] db unreachable after 60s: {host}:{port}", file=sys.stderr)
sys.exit(1)
PY

# ---- 2. alembic upgrade ----
# Idempotent: if head is already current, this is a no-op.
# WHY no `pipenv run`: the runtime image installs the venv at /opt/venv and
# puts it on PATH (see dockerfile.md). pipenv itself is NOT in the runtime
# stage — only the resolved venv is. `flask` and `gunicorn` are available
# directly because /opt/venv/bin is on PATH.
log "running migrations…"
flask db upgrade

# ---- 3. exec gunicorn ----
# WHY exec: replaces the shell so tini's SIGTERM reaches gunicorn directly,
# which then runs --graceful-timeout draining in-flight requests (R112).
log "starting gunicorn on 0.0.0.0:${PORT} (workers=${GUNICORN_WORKERS})…"
exec gunicorn \
    --bind "0.0.0.0:${PORT}" \
    --workers "${GUNICORN_WORKERS}" \
    --timeout "${GUNICORN_TIMEOUT}" \
    --graceful-timeout "${GUNICORN_GRACEFUL_TIMEOUT}" \
    --access-logfile - \
    --error-logfile  - \
    --capture-output \
    "${FLASK_APP}()"
```

## `gunicorn.conf.py` (optional, ship if more knobs needed)

```python
"""Gunicorn config. `graceful_timeout` is the R112 contract."""
import os

bind = f"0.0.0.0:{os.environ.get('PORT', '8000')}"
workers = int(os.environ.get("GUNICORN_WORKERS", "4"))
worker_class = "sync"
timeout = int(os.environ.get("GUNICORN_TIMEOUT", "30"))
graceful_timeout = int(os.environ.get("GUNICORN_GRACEFUL_TIMEOUT", "20"))
accesslog = "-"
errorlog = "-"
capture_output = True
preload_app = False  # WHY False: avoid SQLAlchemy engine fork-safety issues
```

## Checklist (reviewer enforces)
- `set -Eeuo pipefail` at top.
- DB wait loop with a hard deadline (no infinite loop).
- `flask db upgrade` runs BEFORE gunicorn (R111 entrypoint contract).
- `exec gunicorn` (not subshell) so SIGTERM reaches the worker master.
- `--graceful-timeout` set from `GUNICORN_GRACEFUL_TIMEOUT` env (R112).
- NEVER `flask run`, `python -m flask`, or `python app.application` directly
  in production (R111).
- File mode 0755; `chmod +x` in Dockerfile so the COPYed file is executable.
