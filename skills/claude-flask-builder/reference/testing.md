# Testing

## pytest layout
```
api/app/tests/
├── conftest.py             # fixtures
├── unit/                   # pure utils
├── validation/             # schema tests
├── integration/            # endpoint smoke tests
└── e2e/                    # full flow w/ real DB + mocked externals
```

## conftest.py fixtures (must ship)
- `app` — create_application() with test config
- `client` — app.test_client()
- `db_session` — per-test SAVEPOINT rollback
- `test_user` — User factory via secrets-generated creds
- `authenticated_headers` — {token, timezone, platform}
- `mock_stripe` — patches stripe.PaymentIntent/Refund/Transfer/Webhook
- `mock_twilio` — patches Twilio Client
- `mock_fcm` — patches firebase-admin send
- `mock_google_maps` — responses lib URL match
- `mail_outbox` — Flask-Mail.record_messages context

## Must-cover endpoints

**At scaffold time**, only two endpoints exist and BOTH must have tests:
1. POST /api/v1/auth/login
2. GET  /api/v1/health/check

**Reference set** (examples from a mature project — present as feature work
adds them; reviewer/runner FAILs the corresponding R100 check when an
existing endpoint has no test):
- POST /api/v1/order (createOrder — multi-step flow)
- POST /api/v1/order/estimate
- POST /api/v1/order/payment/completed
- GET  /api/v1/order/<id>
- POST /api/v1/order/cancel
- POST /api/v1/account (createAccount with role validation)
- POST /api/v1/payout/creator
- POST /api/v1/file (upload + MIME+size validation)

The runner (Stage 5a) enumerates `app.url_map` at audit time and requires
at least one matching test for every non-static, non-debug route.

## pyproject.toml — pytest config block (scaffolder writes this)
```toml
[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["app/tests"]
addopts = [
    "-ra",
    "--strict-markers",
    "--strict-config",
    "--cov=app",
    "--cov-report=term-missing",
    "--cov-fail-under=60",
]
filterwarnings = [
    "error",
    "ignore::DeprecationWarning:flask_seeder.*",
]
markers = [
    "e2e: full request → DB → response",
    "integration: endpoint with mocked externals",
]
```

## Mocking strategy
- `unittest.mock.patch` for Python-level
- `responses` library for HTTP external calls
- Flask-Mail.record_messages for email assertions

## CI command
```bash
pytest -v --cov=app --cov-report=term-missing --cov-fail-under=60
```

## Hard-to-test areas (documented as known)
- createOrder 10-step chain → integration test with all mocks
- Payout 3-tier split → unit test the math function directly
- Scheduler jobs → call the job function directly (no scheduler simulation)
- RequestInterceptor → autouse fixture disables in unit tests
- AccountSchema @post_dump DB query → refactored to pre-join (R27)
