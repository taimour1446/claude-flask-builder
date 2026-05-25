# Template — Configuration.py

Path: `app/configuration/Configuration.py`. Orchestrator called by
`application.py`'s `create_application()` factory.

Covers rules: R74 (session cookie flags), R90 (global errorhandlers),
R92 (health endpoint returns JSON, not plain string), and the
`app.config["ENV"]` plumbing required by `BaseResponse.respondError`'s
production trace gate.

```python
"""Configuration — orchestrates Flask app wiring. NEVER db.create_all()."""
import logging
from typing import Any

import jwt
from flask import Flask, jsonify
from flask_cors import CORS
from flask_mail import Mail
from marshmallow import ValidationError
from sqlalchemy.exc import IntegrityError, SQLAlchemyError

from app.configuration.Database import db
from app.configuration.Mail import mail
from app.configuration.Scheduler import scheduler
from app.utils.Logging import configure_logging as _configure_logging
from app.utils.Settings import settings
from app.utils.Validation import CustomValidationException

logger = logging.getLogger(__name__)


def configure_logging(app: Flask) -> None:
    """Install JSON formatter in staging/prod, plain stdlib in dev."""
    _configure_logging(app, env=settings.ENV, level=settings.LOG_LEVEL)


def configure_settings(app: Flask) -> None:
    """Validate env (FAIL FAST) and push critical values into app.config."""
    settings.validate()  # raises on missing required
    # WHY mirror into app.config: BaseResponse.respondError reads ENV from
    # current_app.config to gate the dev-only `trace` field (R68).
    app.config["ENV"] = settings.ENV
    app.config["SECRET_KEY"] = settings.SECRET_KEY
    # R74 — session cookie hardening
    app.config["SESSION_COOKIE_SECURE"] = True
    app.config["SESSION_COOKIE_HTTPONLY"] = True
    app.config["SESSION_COOKIE_SAMESITE"] = "Lax"
    # R63 — upper bound for file uploads (image-bomb defense at the edge)
    app.config["MAX_CONTENT_LENGTH"] = settings.MAX_CONTENT_LENGTH_BYTES
    # JSON config
    app.config["JSON_SORT_KEYS"] = False
    app.config["JSONIFY_PRETTYPRINT_REGULAR"] = False


def configure_application(app: Flask) -> None:
    """Cross-cutting middleware (CORS) and global JSON conventions."""
    # R64 — explicit origins from env; NEVER wildcard with credentials
    CORS(
        app,
        resources={r"/api/*": {"origins": settings.CORS_ORIGINS}},
        supports_credentials=True,
        expose_headers=["X-Request-ID"],
    )


def configure_database(app: Flask) -> None:
    """Bind SQLAlchemy to the app. NEVER db.create_all() — Alembic owns schema."""
    app.config["SQLALCHEMY_DATABASE_URI"] = settings.DATABASE_URI
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
    app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
        "pool_pre_ping": True,
        "pool_recycle": 1800,
    }
    db.init_app(app)


def configure_inject(_app: Flask) -> None:
    """DI binder. Empty by default — wire when needed."""
    return None


def configure_mail(app: Flask) -> None:
    app.config["MAIL_SERVER"] = settings.MAIL_SERVER
    app.config["MAIL_PORT"] = settings.MAIL_PORT
    app.config["MAIL_USE_TLS"] = True
    app.config["MAIL_USERNAME"] = settings.MAIL_USERNAME
    app.config["MAIL_PASSWORD"] = settings.MAIL_PASSWORD
    app.config["MAIL_DEFAULT_SENDER"] = settings.MAIL_DEFAULT_SENDER
    mail.init_app(app)


# ---------- R90 — global error handlers ----------

def register_error_handlers(app: Flask) -> None:
    """Catch the canonical exception types so handlers stay clean."""
    from app.utils.BaseResponse import BaseResponse

    @app.errorhandler(CustomValidationException)
    def _custom_validation(exc: CustomValidationException):
        return BaseResponse.respondError(message=str(exc))

    @app.errorhandler(ValidationError)
    def _marshmallow_validation(exc: ValidationError):
        first_field = next(iter(exc.messages))
        first_msg = exc.messages[first_field]
        if isinstance(first_msg, list):
            first_msg = first_msg[0]
        return BaseResponse.respondError(message=str(first_msg))

    @app.errorhandler(IntegrityError)
    def _integrity(_exc: IntegrityError):
        logger.exception("integrity_error")
        return BaseResponse.respondError(message="Duplicate or constraint violation")

    @app.errorhandler(SQLAlchemyError)
    def _sqlalchemy(_exc: SQLAlchemyError):
        logger.exception("sqlalchemy_error")
        return BaseResponse.respondError(message="Database error")

    @app.errorhandler(jwt.ExpiredSignatureError)
    def _jwt_expired(_exc: jwt.ExpiredSignatureError):
        return BaseResponse.respondError(message="Token expired", auth_error=1)

    @app.errorhandler(jwt.InvalidTokenError)
    def _jwt_invalid(_exc: jwt.InvalidTokenError):
        return BaseResponse.respondError(message="Invalid token", auth_error=1)

    @app.errorhandler(404)
    def _not_found(_exc):
        return BaseResponse.respondError(message="Not found")

    @app.errorhandler(Exception)
    def _generic(exc: Exception):
        logger.exception("unhandled_exception")
        # R68: trace ONLY in dev — production returns ticket-id-shaped message
        return BaseResponse.respondError(message="Internal error", trace=str(exc))


# ---------- R92 — health endpoint (JSON, never plain string) ----------

def _register_health(app: Flask) -> None:
    """GET /api/v1/health/check returns JSON {ok, version, deps}."""

    @app.route("/api/v1/health/check", methods=["GET"])
    def health() -> Any:
        deps = {}
        try:
            db.session.execute(db.text("SELECT 1"))
            deps["db"] = "ok"
        except SQLAlchemyError:
            deps["db"] = "down"
        body = {"ok": all(v == "ok" for v in deps.values()),
                "version": settings.APP_VERSION,
                "env": settings.ENV,
                "deps": deps}
        return jsonify(body), 200 if body["ok"] else 503


def register_blueprints(app: Flask) -> None:
    """Register every blueprint under /api/v1. Order does not matter."""
    from app.api import (  # noqa: PLC0415 — late import to avoid model-class-load cycles
        account_blueprint,
        # ... add new blueprint factories here ...
    )
    _register_health(app)
    app.register_blueprint(account_blueprint(), url_prefix="/api/v1")
    # app.register_blueprint(<x>_blueprint(), url_prefix="/api/v1")


def register_scheduler_jobs(app: Flask) -> None:
    """Initialize APScheduler with the app context."""
    scheduler.init_app(app)
    scheduler.start()
    # Import job modules so their @scheduler.task decorators run.
    from app import scheduler_jobs  # noqa: F401, PLC0415
```

## Checklist (reviewer enforces)
- ALL three session cookie flags set in `configure_settings` (R74).
- CORS origins from env, supports_credentials=True with explicit origins
  (not `*`) (R64).
- `db.init_app(app)`; NEVER `db.create_all()` (architecture rule).
- `register_error_handlers` covers: CustomValidationException,
  ValidationError, IntegrityError, SQLAlchemyError, jwt.ExpiredSignatureError,
  jwt.InvalidTokenError, 404, generic Exception (R90).
- Health endpoint returns `jsonify(...)` with `{ok, version, env, deps}` —
  NEVER `return "ok"` (R92).
- `app.config["ENV"]` mirrored from `settings.ENV` (lets BaseResponse gate
  trace in prod).
- `MAX_CONTENT_LENGTH` set (R63).
- Late blueprint imports avoid model-load cycles.
