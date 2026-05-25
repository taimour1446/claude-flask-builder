# Template — .env.example

Ship at project root. NEVER commit a `.env` with real values (gitignored).

```
# === Core ===
ENV=dev                              # dev | staging | prod
FLASK_APP=app.application:create_application
SECRET_KEY=replace-with-output-of-python-c-import-secrets-print-secrets-token-urlsafe-32  # ≥32 raw bytes; Settings.validate() rejects shorter
DATABASE_URI=postgresql+psycopg2://user:pass@localhost:5432/dbname?sslmode=require
BASE_URL=https://api.example.com
INTERNAL_BASE_URL=http://localhost:8000  # P6-2

# === JWT ===
JWT_ACCESS_TTL_MINUTES=30
JWT_REFRESH_TTL_DAYS=30

# === Mail (Flask-Mail) ===
MAIL_SERVER=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_DEFAULT_SENDER=noreply@example.com

# === Stripe (if enabled) ===
STRIPE_SECRET_KEY=                                  # sk_live_... or sk_test_...
STRIPE_PUBLISHABLE_KEY=                             # pk_live_... or pk_test_...
STRIPE_WEBHOOK_SECRET=                              # whsec_...
STRIPE_RETURN_URL_ALLOWLIST=https://app.example.com/payment/return,https://app.example.com/payment/cancel
PLATFORM_ACCOUNT_ID=

# === Twilio (if enabled) ===
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=          # corrected from TOEKN
TWILIO_FROM_NUMBER=+1...    # P15-10 — env, not hardcoded

# === FCM (if enabled) ===
FCM_SERVER_KEY=

# === AWS S3 (if enabled) ===
S3_BUCKET=
AWS_ACCESS_KEY=
AWS_ACCESS_SECRET=

# === GCS (if enabled) ===
GCS_BUCKET=
GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa.json

# === Google Maps + TollGuru (if enabled) ===
GOOGLE_API_KEY=
TOLLGURU_API_KEY=
TOLLGURU_URL=https://prod.tollguru.com    # P22-7 — env, not hardcoded

# === Observability ===
SENTRY_DSN=
LOG_LEVEL=INFO

# === CORS ===
CORS_ORIGINS=https://app.example.com,https://admin.example.com

# === Cron secret (replaces secret-path-auth) ===
CRON_SHARED_SECRET=
```
