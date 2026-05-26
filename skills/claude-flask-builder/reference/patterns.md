# Signature Patterns

## 1. BaseResponse envelope (LOCKED for client compat)
Every JSON response: `{status, data, error, authError, force_update, message, request_id}`
- `respond(data, message?)` → error=0, status=200
- `respondError(message, authError=0, force_update=0)` → error=1, status=200
- `request_id` populated by RequestInterceptor's X-Request-ID
- `trace` field ONLY in dev — never in production response

### Full implementation skeleton (utils/BaseResponse.py)
```python
"""BaseResponse — the locked JSON envelope. NEVER change shape; clients depend on it."""
from typing import Any
from flask import jsonify, g, current_app


class BaseResponse:
    """Two static constructors. Both return a Flask Response."""

    @staticmethod
    def respond(data: Any = None, message: str = "") -> tuple:
        """Success envelope. status=200, error=0."""
        return jsonify({
            "status": 200,
            "error": 0,
            "authError": 0,
            "force_update": 0,
            "message": message,
            "data": data,
            "request_id": getattr(g, "request_id", None),
        }), 200

    @staticmethod
    def respondError(message: str = "Internal error", auth_error: int = 0,
                     force_update: int = 0, trace: str | None = None) -> tuple:
        """Error envelope. status=200 (client expects 200; reads `error`+`message`)."""
        body = {
            "status": 200,
            "error": 1,
            "authError": auth_error,
            "force_update": force_update,
            "message": message,
            "data": None,
            "request_id": getattr(g, "request_id", None),
        }
        # R68: trace ONLY in dev — production returns ticket id only.
        if trace and current_app.config.get("ENV") != "production":
            body["trace"] = trace
        return jsonify(body), 200
```

## 2. Controller skeleton (R23)
```python
@blueprint.route('/path', methods=['POST'])
@token_required  # if auth required
def handler(current_user) -> BaseResponse:
    try:
        data = XService.method(request, current_user)
        return BaseResponse.respond(data)
    except CustomValidationException as e:
        return BaseResponse.respondError(message=str(e))
    except Exception as e:
        logger.exception('handler_failed')
        return BaseResponse.respondError(message='Internal error')
```
Public routes omit `@token_required` AND `current_user` param.
Raw returns (files/exports/health) skip BaseResponse — sanctioned exception.

## 3. Service skeleton (R24)
```python
class XService:
    @staticmethod
    def method(request, current_user=None):
        data = Validation.validate(request, CreateXValidation())
        Auth.assert_role(current_user, [Role.Admin])  # if role-gated
        with db.session.begin_nested():  # SAVEPOINT for multi-model
            obj = X.create(data)
            Y.create(...)
        return XSchema().dump(obj)
```
Service composes multi-model writes in ONE transaction (R25).
Service NEVER touches request beyond passing to Validation.validate.

## 4. Validation schema (R26, R28)
```python
class CreateXValidation(Schema):
    name = fields.String(required=True, error_messages={'required': 'Name is required'})

    class Meta:
        unknown = EXCLUDE  # not INCLUDE — R62

    @validates_schema
    def custom(self, data, **kwargs):
        if X.exists(data['name']):
            raise CustomValidationException(Constants.NameTaken)
```

## 5. DTO schema (R27)
```python
class XSchema(Schema):
    id = fields.Int()
    name = fields.Str()

    @post_dump
    def derive(self, data, **kwargs):
        # NO DB queries here — pre-join in the service
        if data == {}:
            return None
        data['full_name'] = f"{data.get('first_name', '')} {data.get('last_name', '')}".strip()
        return data
```

## 6. Model skeleton (R25, R43, R44)
```python
class X(db.Model):
    __tablename__ = 'xs'  # plural
    id = Column(Integer, primary_key=True)
    name = Column(Text, nullable=False)
    amount = Column(Numeric(10, 2), nullable=False, default=0)  # R45
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    deleted_at = Column(DateTime(timezone=True), nullable=True)  # R47

    def to_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

    @staticmethod
    def create(data):
        obj = X(name=data['name'], amount=data.get('amount', 0))  # explicit fields, no __dict__.update
        db.session.add(obj)
        db.session.commit()
        return obj

    @staticmethod
    def get_by_id(id):
        return db.session.query(X).filter_by(id=id, deleted_at=None).first()
```

## 7. JWT auth (R60)
- `Auth.generate_tokens(user)` returns access + refresh tokens
- Access TTL 30 min, refresh 30 days, rotation on use (refresh_jti rotated)
- `@token_required` decorator validates exp, loads user, injects `current_user` kwarg

### Full implementation skeleton (utils/Auth.py)
```python
"""Auth — JWT generate, decode, token_required + role_required decorators."""
import datetime as dt
import logging
from functools import wraps
from typing import Callable

import jwt
from flask import request

from app.utils.BaseResponse import BaseResponse
from app.utils.Settings import settings
from app.model.User import User

logger = logging.getLogger(__name__)


class Auth:
    @staticmethod
    def generate_tokens(user: User) -> dict:
        """Issue access + refresh. Both include `exp` (R60)."""
        now = dt.datetime.now(dt.timezone.utc)
        access = jwt.encode(
            {"sub": user.id, "type": "access",
             "exp": now + dt.timedelta(minutes=settings.JWT_ACCESS_TTL_MINUTES)},
            settings.SECRET_KEY, algorithm="HS256",
        )
        refresh = jwt.encode(
            {"sub": user.id, "type": "refresh",
             "exp": now + dt.timedelta(days=settings.JWT_REFRESH_TTL_DAYS),
             "jti": user.refresh_jti},  # rotation: rotate jti on each refresh
            settings.SECRET_KEY, algorithm="HS256",
        )
        return {"access": access, "refresh": refresh}

    @staticmethod
    def assert_role(current_user: User, allowed: list[str]) -> None:
        """Service-layer role gate (R72)."""
        if current_user.role not in allowed:
            raise PermissionError("role not authorized")


def token_required(fn: Callable) -> Callable:
    """Decorator: validates access token, loads user, injects current_user."""
    @wraps(fn)
    def wrapper(*args, **kwargs):
        token = request.headers.get("Authorization", "").removeprefix("Bearer ").strip()
        if not token:
            return BaseResponse.respondError(message="Missing token", auth_error=1)
        try:
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            return BaseResponse.respondError(message="Token expired", auth_error=1)
        except jwt.InvalidTokenError:
            return BaseResponse.respondError(message="Invalid token", auth_error=1)
        if payload.get("type") != "access":
            return BaseResponse.respondError(message="Wrong token type", auth_error=1)
        user = User.get_by_id(payload["sub"])
        if user is None:
            return BaseResponse.respondError(message="User not found", auth_error=1)
        return fn(*args, current_user=user, **kwargs)
    return wrapper
```

## 8. (reserved)

Round 1–4 had a duplicate RequestInterceptor stub here; Round 5 removed it
because the full skeleton lives at §11. The slot is left numbered as
"reserved" so cross-references elsewhere (`scaffold-checklist.md` cites
`patterns.md §1/§7/§9/§10/§11`) remain stable. Do not renumber — adding a
new section here is fine if it stays self-contained.

## 10. Validation.validate (utils/Validation.py)
Universal request-validator. Services call ONLY this — never `Schema().load(...)` directly.

```python
"""Validation — single entry point. Raises CustomValidationException on failure."""
from flask import Request
from marshmallow import Schema, ValidationError


class CustomValidationException(Exception):
    """Validation/uniqueness failures. Controllers translate to BaseResponse.respondError."""


class Validation:
    @staticmethod
    def validate(req: Request, schema: Schema) -> dict:
        """Load + validate JSON body. Returns the cleaned dict.

        :raises CustomValidationException: any Marshmallow error, first-message-wins.
        """
        try:
            return schema.load(req.get_json(silent=True) or {})
        except ValidationError as exc:
            # WHY first message: the BaseResponse envelope carries a single
            # human-readable string. The validation schema's `error_messages`
            # is the source of truth for wording (R28).
            first_field = next(iter(exc.messages))
            first_msg = exc.messages[first_field]
            if isinstance(first_msg, list):
                first_msg = first_msg[0]
            raise CustomValidationException(first_msg) from exc
```

## 11. RequestInterceptor (utils/RequestInterceptor.py)
Wraps every request: assigns request_id, logs redacted body, attaches header.

```python
"""RequestInterceptor — X-Request-ID + redacted-body logging (R67, R91)."""
import logging
import secrets
from flask import Flask, g, request
from werkzeug.exceptions import BadRequest

logger = logging.getLogger(__name__)

# WHY allowlist: drift-resistant — adding a new field elsewhere never leaks
# secrets through logs by default (R67).
_REDACT_FIELDS: frozenset[str] = frozenset({
    # Auth secrets
    "password", "new_password", "current_password",
    "ptoken", "otp", "token", "refresh_token", "access_token",
    # Payment card data (PCI scope)
    "card_number", "cvc", "cvv", "credit_card",
    # API + provider secrets
    "stripe_secret", "stripe_secret_key", "api_key", "secret_key",
    "authorization", "cookie",
    # PII — GDPR/HIPAA-shaped fields. Redacted by default; remove if your
    # downstream log pipeline already encrypts at rest AND you need the
    # value for debugging.
    "email", "phone", "phone_number", "ssn", "national_id",
    "date_of_birth", "dob",
})


def _redact(payload: dict | list | None) -> dict | list | None:
    """Recursive case-insensitive redaction. Replaces matched values with ***."""
    if isinstance(payload, dict):
        # WHY .lower(): redaction key list is canonical lowercase; payloads
        # may arrive with mixed case from third-party callers. We never want
        # a Stripe_Secret_Key or PASSWORD slip through (R67).
        return {
            k: ("***" if k.lower() in _REDACT_FIELDS else _redact(v))
            for k, v in payload.items()
        }
    if isinstance(payload, list):
        return [_redact(v) for v in payload]
    return payload


class RequestInterceptor:
    @staticmethod
    def intercept(app: Flask) -> None:
        """Register before/after hooks."""

        @app.before_request
        def _before() -> None:
            # WHY url-safe 16 bytes: roughly UUID-strength but URL/header safe.
            g.request_id = request.headers.get("X-Request-ID") or secrets.token_urlsafe(16)
            # WHY (TypeError, ValueError, BadRequest): the only exceptions
            # `request.get_json(silent=True)` can raise pass through these.
            # We must NOT let log preparation crash the request (R17 allows
            # specific exceptions; this is NOT a bare except).
            try:
                body = request.get_json(silent=True)
            except (TypeError, ValueError, BadRequest):
                body = None
            # WHY redact query args too: secrets often slip into URLs
            # (e.g. `GET /reset?ptoken=...`). Access logs + reverse-proxy
            # logs would otherwise capture them in plaintext.
            redacted_args = _redact(dict(request.args.to_dict(flat=True))) or {}
            logger.info("request_in", extra={
                "request_id": g.request_id, "method": request.method,
                "path": request.path, "args": redacted_args,
                "body": _redact(body),
            })

        @app.after_request
        def _after(response):
            response.headers["X-Request-ID"] = getattr(g, "request_id", "")
            logger.info("request_out", extra={
                "request_id": getattr(g, "request_id", None),
                "status": response.status_code,
            })
            return response
```

## 9. Service-direct cron jobs (P6-8)
NO self-callback `requests.post(BASE_URL + ...)` pattern.
Cron jobs in `scheduler_jobs/` call services directly:
```python
@scheduler.task('cron', id='purge_pending_orders', hour=0, timezone='UTC', max_instances=1)
def purge_pending_orders():
    with scheduler.app.app_context():
        with distributed_lock('purge_pending_orders'):
            OrderService.purge_pending(...)
```

### `distributed_lock` (utils/distributed_lock.py) — Postgres advisory lock
```python
"""distributed_lock — Postgres advisory-lock context manager.

WHY advisory lock (not Redis): the app already has a Postgres connection;
adding Redis is a new dependency outside the Locked Stack (R141). Postgres
advisory locks are per-connection, automatically released on disconnect,
and survive multi-pod deploys.
"""
import contextlib
import hashlib
import logging
from typing import Iterator

from sqlalchemy import text

from app.configuration.Database import db

logger = logging.getLogger(__name__)


def _lock_id(name: str) -> int:
    """Deterministic int64 id from the lock name (Postgres pg_try_advisory_lock arg)."""
    h = hashlib.sha256(name.encode("utf-8")).digest()
    # int64 range; signed two's-complement
    val = int.from_bytes(h[:8], "big", signed=True)
    return val


@contextlib.contextmanager
def distributed_lock(name: str) -> Iterator[bool]:
    """Try-acquire a named advisory lock. Yields True if acquired, False if not.

    Callers MUST check the yielded value when they care about exclusivity:
        with distributed_lock('purge') as acquired:
            if not acquired:
                return  # another pod is doing it
            ...

    For cron jobs marked `max_instances=1` AND using this lock, double-run
    is impossible even across pods.
    """
    lock_id = _lock_id(name)
    acquired = bool(db.session.execute(
        text("SELECT pg_try_advisory_lock(:k)"), {"k": lock_id}
    ).scalar())
    try:
        yield acquired
    finally:
        if acquired:
            db.session.execute(
                text("SELECT pg_advisory_unlock(:k)"), {"k": lock_id}
            )
            db.session.commit()
```
