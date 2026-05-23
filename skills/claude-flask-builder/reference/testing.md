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

## 10 must-cover endpoints
1. POST /api/v1/auth/login
2. POST /api/v1/order (createOrder — 10-step flow)
3. POST /api/v1/order/estimate
4. POST /api/v1/order/payment/completed
5. GET /api/v1/order/<id>
6. POST /api/v1/order/cancel
7. POST /api/v1/account (createAccount with role validation)
8. POST /api/v1/payout/creator
9. GET /api/v1/health/check
10. POST /api/v1/file (upload + MIME+size validation)

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
