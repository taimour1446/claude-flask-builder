# External API Integrations

## Stripe
- `stripe.api_key = settings.STRIPE_SECRET` at module load via Settings, not os.getenv per call
- ALWAYS pass `idempotency_key=` on mutations (PaymentIntent.create, Refund.create, Transfer.create)
- Amounts: `math.ceil(Decimal(amount) * 100)` (cents, never Float)
- Webhook handler: `stripe.Webhook.construct_event(payload, sig_header, settings.STRIPE_WEBHOOK_SECRET)`
- Catch `stripe.error.StripeError` specifically — not `Exception`

## Twilio
- USE the twilio SDK, not hand-rolled requests (reference used hand-rolled with no timeout)
- `Client(account_sid, auth_token).messages.create(to=..., from_=settings.TWILIO_FROM, body=...)`
- Catch `TwilioRestException`

## FCM
- USE firebase-admin SDK (not raw HTTP to fcm.googleapis.com — legacy)
- On 'Not Registered'/'InvalidRegistration' → set user.device_token = NULL

## Google Maps
- API key in HEADER not URL (referrer leak)
- Distance Matrix / Directions / Places: env-driven URLs
- Cache results when possible (TTL 1 hour for same route)

## TollGuru
- Try/except wrapper with graceful fallback (`toll_fee=0`, mark `toll_unavailable=True`)

## Boto3 S3
- IAM role preferred; env creds fallback
- `ACL='private'` default (R71); presigned URLs for sharing
- `Bucket(...).Object(key).upload_fileobj(file, ExtraArgs={'ContentType': ..., 'ServerSideEncryption': 'AES256'})`

## GCS
- Workload Identity preferred; key file via GOOGLE_APPLICATION_CREDENTIALS fallback

## Flask-Mail
- DKIM/SPF guidance in README
- HTML + plain-text alternative
- Base email template includes unsubscribe link + accessibility (ARIA)

## Universal rules
- Every requests call: `timeout=(5, 30)` (connect, read)
- Per-provider `requests.Session()` singleton (R84)
- Wrap response.json() in try/except for JSONDecodeError (R82)
- Centralize each integration in `app/integrations/<provider>.py`
