# Security — the corrections list

The reference codebase had 50+ security defects. The skill ships them ALL fixed
and the reviewer ENFORCES the fixes (R60–R74 in `coding-standards.md`).

## What the skill enforces (vs reference)

| Defect in reference | Skill fix |
| --- | --- |
| `.env.local` with real Stripe/AWS/Twilio/Mail/FCM secrets committed | Only `.env.example` (placeholders), `.env*` gitignored except example |
| `text("... " + value + " ...")` / `.format()` in 6+ raw SQL sites | Every `text()` uses `:bindparam` |
| JWT no `exp` claim — tokens valid forever | `exp` claim required, 15-60 min TTL, refresh-token rotation |
| `random.randrange` for passwords/PINs/tokens | `secrets.token_urlsafe(32)` |
| `model.__dict__.update(data)` allows mass-assignment of `is_owner`, `role` | Explicit field assignment from validated dict only |
| Marshmallow `unknown=INCLUDE` propagates unknown fields into models | `unknown=EXCLUDE` (or RAISE) for request validation |
| File upload no MIME / size / image-bomb check | Allowlist + `MAX_CONTENT_LENGTH` + `Image.MAX_IMAGE_PIXELS` |
| CORS wildcard + credentials | Explicit `origins=[...]` from env |
| Password reset token = 10 lowercase chars | 32-byte `secrets.token_urlsafe` + 15-min TTL + single-use |
| OTP no expiry, no rate limit | 5-min TTL + max-5-attempts-per-hour |
| Secret-path cron auth (`/HGSJDFKNSKHER`) | HMAC-signed header OR drop the self-callback entirely (direct service call) |
| `traceback.format_exc()` in response body | Dev-only; prod returns ticket id only |
| FAQ `\|safe` filter on user content | bleach-sanitized HTML |
| Flask session: no secure/httponly/samesite | All three set |
| `os.getenv()` everywhere, no validation | Centralized `Settings` class validates on startup |
| Hardcoded URLs (TollGuru dev URL, Twilio +1612... number) | Env-driven |
| S3 ACL=public-read default | `private` default + presigned URLs |
| Stripe webhook no signature verification | `stripe.Webhook.construct_event` required |
| Stripe transfer/charge no idempotency_key | Required on every mutation |
| FAQService admin routes accept any authed user | Role assertion in service |
| `/order/live/track/<token>` decodes base64 (forgeable) | Signed JWT tracking token |
| RequestInterceptor logs full body incl passwords | Redaction list for password/card/token/otp |
| `force_update` flag half-wired | Either fully implement version gate OR remove |
| No X-Request-ID propagation | Header set on every response |
| Bare `except:` swallows everything | Specific exceptions only (reviewer fails bare) |

## What the scaffolder writes by default
- `Settings` class validates required env vars at startup, FAILS FAST.
- `Logging.py` configures structured stdlib `logging` (JSON formatter in prod).
- Stripe webhook handler with signature verification skeleton (if Stripe enabled).
- Idempotency-key helper for all Stripe mutations.
- Token-redaction map in `RequestInterceptor`.
- Single-use ptoken + 15-min TTL + DB `ptoken_expires_at` column.

## Reviewer FAILS on first sight
- Any `text("... {x} ...")` / f-string in raw SQL.
- Any `model.__dict__.update(data)`.
- Any `random` for security tokens.
- Any `Float` column for money.
- Any `print(` in src.
- Any `bare except:`.
- Any literal-looking secret (hex >32 chars, looks like a key) in source.
- Missing `timeout=` on `requests.*`.
- Missing `idempotency_key=` on Stripe mutations.
- `traceback.format_exc()` in a respondError when `ENV=production`.
