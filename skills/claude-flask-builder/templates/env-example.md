# Template — .env.example

Ship at project root. NEVER commit a `.env` with real values (gitignored).

```
# === Core ===
ENV=dev                              # dev | staging | prod
FLASK_APP=app.application:create_application
SECRET_KEY=change-me-secrets-token_urlsafe-32
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
STRIPE_SECRET=
STRIPE_PUBLIC_KEY=
STRIPE_WEBHOOK_SECRET=
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
