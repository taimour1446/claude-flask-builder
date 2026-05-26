# Template — conftest.py (pytest fixtures)

Path: `app/tests/conftest.py`. Loaded automatically by pytest from
`pyproject.toml`'s `[tool.pytest.ini_options].testpaths = ["app/tests"]`.

This is the **mandatory** fixture surface. Every name listed in
`testing.md` "conftest.py fixtures (must ship)" is defined here. New
tests pick from this surface — they do NOT invent fixtures with the same
name in test files.

```python
"""conftest.py — fixtures shared across unit/validation/integration/e2e tests.

Lifecycle:
- `app` (session): one Flask app per test session, test-config env vars set.
- `db_session` (function): each test gets its own SAVEPOINT, rolled back
  after the test — gives true isolation without dropping/recreating tables.
- `client` (function): app.test_client() pinned to the rolled-back session.
"""
import os
import secrets
from collections.abc import Iterator
from unittest.mock import MagicMock, patch

import bcrypt
import pytest
from flask import Flask
from flask.testing import FlaskClient
from sqlalchemy.orm import sessionmaker

# ---- Test-only env (set BEFORE create_application is imported) ----
# WHY here: Settings.validate() runs at app boot. Required env must be
# in os.environ before the import.
os.environ.setdefault("ENV", "dev")
os.environ.setdefault("APP_NAME", "test-app")
os.environ.setdefault("APP_VERSION", "0.0.0-test")
os.environ.setdefault("SECRET_KEY", "x" * 32)  # ≥32 raw bytes for Settings.validate
os.environ.setdefault("DATABASE_URI", os.environ.get("TEST_DATABASE_URI", "sqlite:///:memory:"))
os.environ.setdefault("CORS_ORIGINS", "http://localhost:3000")

from app.application import create_application  # noqa: E402 — env-before-import
from app.configuration.Database import db as _db  # noqa: E402
from app.model.User import User  # noqa: E402
from app.utils import Constants  # noqa: E402


# ============================================================================
# App + DB
# ============================================================================

@pytest.fixture(scope="session")
def app() -> Iterator[Flask]:
    """Boot the app once per test session."""
    app = create_application()
    app.config["TESTING"] = True
    with app.app_context():
        _db.create_all()  # tests use a throwaway DB; production uses Alembic
        yield app
        _db.session.remove()
        _db.drop_all()


@pytest.fixture(scope="function")
def db_session(app: Flask) -> Iterator:
    """Per-test SAVEPOINT rollback.

    WHY nested SAVEPOINT: gives real transactional isolation without
    truncating tables between tests. Every commit inside the test is
    nested under this SAVEPOINT and reversed at teardown.
    """
    with app.app_context():
        connection = _db.engine.connect()
        transaction = connection.begin()
        Session = sessionmaker(bind=connection)
        session = Session()
        _db.session = session  # type: ignore[assignment]

        nested = connection.begin_nested()
        try:
            yield session
        finally:
            nested.rollback()
            session.close()
            transaction.rollback()
            connection.close()


@pytest.fixture(scope="function")
def client(app: Flask, db_session) -> FlaskClient:
    """Test client bound to the rolled-back session."""
    return app.test_client()


# ============================================================================
# Users + auth
# ============================================================================

@pytest.fixture(scope="function")
def test_user(db_session) -> User:
    """Create a baseline customer user with a known-strong password.

    Password comes from secrets — never hard-coded — and is set via the
    User.password hybrid setter (triggers bcrypt). Returns the persisted
    User instance.
    """
    plaintext = secrets.token_urlsafe(24)
    user = User(
        email=f"test-{secrets.token_hex(4)}@example.com",
        phone=f"+1415555{secrets.randbelow(10000):04d}",
        role=Constants.Role.CUSTOMER.value,
    )
    user.password = plaintext  # bcrypt setter on User
    user._plaintext_password = plaintext  # type: ignore[attr-defined] — for the test only
    db_session.add(user)
    db_session.commit()
    return user


@pytest.fixture(scope="function")
def test_admin(db_session) -> User:
    """Admin user — for tests that exercise R72 role gates."""
    plaintext = secrets.token_urlsafe(24)
    user = User(
        email=f"admin-{secrets.token_hex(4)}@example.com",
        phone=f"+1415666{secrets.randbelow(10000):04d}",
        role=Constants.Role.ADMIN.value,
    )
    user.password = plaintext
    user._plaintext_password = plaintext  # type: ignore[attr-defined]
    db_session.add(user)
    db_session.commit()
    return user


@pytest.fixture(scope="function")
def authenticated_headers(test_user: User) -> dict:
    """Bearer + standard headers a controller expects."""
    from app.utils.Auth import Auth
    tokens = Auth.generate_tokens(test_user)
    return {
        "Authorization": f"Bearer {tokens['access']}",
        "X-Timezone": "UTC",
        "X-Platform": "test",
    }


# ============================================================================
# External-service mocks (R85: vendor SDKs are mocked at the SDK boundary)
# ============================================================================

@pytest.fixture(scope="function")
def mock_stripe() -> Iterator[MagicMock]:
    """Patch Stripe SDK at module level.

    Tests assert on `mock_stripe.PaymentIntent.create.call_args.kwargs`
    to verify idempotency_key + amount + currency.
    """
    with patch("stripe.PaymentIntent") as pi, \
         patch("stripe.Refund") as refund, \
         patch("stripe.Transfer") as transfer, \
         patch("stripe.Webhook") as webhook:
        m = MagicMock()
        m.PaymentIntent = pi
        m.Refund = refund
        m.Transfer = transfer
        m.Webhook = webhook
        yield m


@pytest.fixture(scope="function")
def mock_twilio() -> Iterator[MagicMock]:
    """Patch twilio.rest.Client."""
    with patch("twilio.rest.Client") as cli:
        cli.return_value.messages.create.return_value = MagicMock(sid="SM_test")
        yield cli


@pytest.fixture(scope="function")
def mock_fcm() -> Iterator[MagicMock]:
    """Patch firebase_admin messaging.send."""
    with patch("firebase_admin.messaging.send") as send:
        send.return_value = "test-message-id"
        yield send


@pytest.fixture(scope="function")
def mock_responses() -> Iterator:
    """`responses` library for matching arbitrary HTTP."""
    import responses as _responses
    with _responses.RequestsMock() as r:
        yield r


@pytest.fixture(scope="function")
def mail_outbox(app: Flask) -> Iterator[list]:
    """Capture Flask-Mail sends without delivering."""
    from app.configuration.Mail import mail
    with mail.record_messages() as outbox:
        yield outbox
```

## Checklist (reviewer enforces)
- `os.environ.setdefault` calls happen BEFORE `from app.application import
  create_application` so `Settings.validate()` sees test values.
- `db_session` uses nested SAVEPOINT + rollback — never `db.drop_all`
  between tests.
- `test_user` / `test_admin` use `secrets.*` for password generation
  (R20).
- Mocks patch at the SDK boundary (`stripe.PaymentIntent`, `twilio.rest.
  Client`, `firebase_admin.messaging.send`) — NOT at our service layer.
- `mail_outbox` uses Flask-Mail's `record_messages()` — no SMTP in tests.
- `authenticated_headers` calls `Auth.generate_tokens(user)` (the real
  generator) — keeps tests honest about the JWT shape.
