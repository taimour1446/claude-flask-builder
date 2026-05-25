# Template — Integration Client (external HTTP API)

Path: `app/integrations/<Provider>Client.py`. One file per provider (Stripe,
Twilio, FCM, S3, GCS, GoogleMaps, TollGuru, …).

Covers rules: R17 (specific except), R18 (logger not print), R20 (secrets only
when generating idempotency keys, never `random`), R80 (timeout=), R82
(`response.json()` try/except), R83 (no hardcoded URLs — env-driven via
`Settings`), R84 (`requests.Session()` per provider, module singleton), R85
(prefer vendor SDK when one exists).

For Stripe specifically: also covers R70 (webhook signature verification —
shown in `webhook.py.md`) and R81 (`idempotency_key=` on every mutation).

```python
"""<Provider>Client — singleton wrapper for the <Provider> external API.

WHY singleton: one `requests.Session` per provider reuses TCP connections and
ensures every call inherits the same retry + timeout policy (R84).
"""
import logging
import secrets
from json import JSONDecodeError
from typing import Any

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

from app.utils.Settings import settings

logger = logging.getLogger(__name__)

# WHY module-level: enforces singleton per worker process (R84).
_session: requests.Session | None = None

# WHY 10s: balance between user-facing latency and provider tail latency.
# Override per-call when a known-slow endpoint requires more.
_DEFAULT_TIMEOUT_SECONDS: int = 10


def _build_session() -> requests.Session:
    """Construct the provider session with retry policy.

    WHY Retry on connect/read but NOT on POST 5xx by default: avoid double-
    submitting non-idempotent writes. Per-call code adds `idempotency_key=`
    when retry-on-write is needed (R81).
    """
    session = requests.Session()
    retry = Retry(
        total=3,
        connect=3,
        read=2,
        backoff_factor=0.5,
        status_forcelist=(502, 503, 504),
        allowed_methods=frozenset({"GET", "HEAD", "OPTIONS"}),
        raise_on_status=False,
    )
    adapter = HTTPAdapter(max_retries=retry, pool_connections=10, pool_maxsize=20)
    session.mount("https://", adapter)
    session.mount("http://", adapter)
    session.headers.update({"User-Agent": f"{settings.APP_NAME}/{settings.APP_VERSION}"})
    return session


def _get_session() -> requests.Session:
    """Lazy-init the singleton on first call."""
    global _session
    if _session is None:
        _session = _build_session()
    return _session


def _new_idempotency_key() -> str:
    """Generate a write-safe idempotency key.

    WHY secrets.token_urlsafe: cryptographic randomness — `random` is
    predictable and unfit for keys that protect against duplicate charges (R20).
    """
    return secrets.token_urlsafe(24)


class <Provider>ApiError(Exception):
    """Raised when the provider returns an unrecoverable error.

    WHY a domain exception (not bare `requests.RequestException`): callers in
    the service layer can catch this specific type without leaking the
    transport-layer exception hierarchy across boundaries (R17).
    """


class <Provider>Client:
    """Thin wrapper around the <Provider> REST API."""

    # WHY classvar from Settings: no hardcoded URLs (R83).
    _BASE_URL: str = settings.<PROVIDER>_BASE_URL
    _API_KEY: str = settings.<PROVIDER>_API_KEY

    @staticmethod
    def fetch_resource(resource_id: str) -> dict[str, Any]:
        """Read a single resource.

        :raises <Provider>ApiError: network failure, non-2xx, or invalid JSON.
        """
        url = f"{<Provider>Client._BASE_URL}/resources/{resource_id}"
        try:
            response = _get_session().get(
                url,
                headers={"Authorization": f"Bearer {<Provider>Client._API_KEY}"},
                timeout=_DEFAULT_TIMEOUT_SECONDS,  # R80
            )
        except requests.RequestException as exc:
            logger.exception("provider.get failed", extra={"url": url})
            raise <Provider>ApiError("network failure") from exc

        if response.status_code >= 400:
            logger.warning(
                "provider.get non-2xx",
                extra={"url": url, "status": response.status_code},
            )
            raise <Provider>ApiError(f"provider returned {response.status_code}")

        try:
            return response.json()  # R82 — wrapped immediately below
        except (JSONDecodeError, ValueError) as exc:
            logger.exception("provider.get invalid JSON", extra={"url": url})
            raise <Provider>ApiError("invalid JSON in provider response") from exc

    @staticmethod
    def create_resource(payload: dict[str, Any], idempotency_key: str | None = None) -> dict[str, Any]:
        """Write a resource. ALWAYS sends an idempotency key (R81 equivalent for non-Stripe).

        :param idempotency_key: caller-provided to deduplicate retries; auto-
            generated if omitted. Persist the key on the originating record
            so reruns of the same user action reuse the same key.
        """
        key = idempotency_key or _new_idempotency_key()
        url = f"{<Provider>Client._BASE_URL}/resources"
        try:
            response = _get_session().post(
                url,
                json=payload,
                headers={
                    "Authorization": f"Bearer {<Provider>Client._API_KEY}",
                    "Idempotency-Key": key,
                },
                timeout=_DEFAULT_TIMEOUT_SECONDS,  # R80
            )
        except requests.RequestException as exc:
            logger.exception("provider.create failed", extra={"url": url, "key": key})
            raise <Provider>ApiError("network failure") from exc

        if response.status_code >= 400:
            logger.warning(
                "provider.create non-2xx",
                extra={"url": url, "status": response.status_code, "key": key},
            )
            raise <Provider>ApiError(f"provider returned {response.status_code}")

        try:
            return response.json()  # R82
        except (JSONDecodeError, ValueError) as exc:
            logger.exception("provider.create invalid JSON", extra={"url": url, "key": key})
            raise <Provider>ApiError("invalid JSON in provider response") from exc
```

## Stripe mutation snippet (drop-in for `StripeClient.py`)

Covers R81 (`idempotency_key=`) and R69 (env-allowlist redirect validation
when the call produces a `return_url`):

```python
import stripe
from app.utils.Settings import settings

stripe.api_key = settings.STRIPE_API_KEY


def create_payment_intent(amount_cents: int, currency: str, customer_id: str,
                          return_url: str, idempotency_key: str) -> stripe.PaymentIntent:
    """Create a Stripe PaymentIntent.

    :raises ValueError: return_url not in allowlist (R69).
    :raises stripe.error.StripeError: provider failure (caller decides).
    """
    if not any(return_url.startswith(p) for p in settings.STRIPE_RETURN_URL_ALLOWLIST):
        raise ValueError("return_url not in allowlist")
    return stripe.PaymentIntent.create(
        amount=amount_cents,
        currency=currency,
        customer=customer_id,
        return_url=return_url,
        idempotency_key=idempotency_key,  # R81 — REQUIRED on every mutation
    )
```

## Checklist (reviewer enforces)
- Module-level `requests.Session()` singleton (R84).
- `timeout=` on every `.get/.post/.put/.delete/.patch/.request` (R80).
- `.json()` wrapped in try/except `JSONDecodeError` (R82).
- Base URL + API key from `settings` — never hardcoded (R83).
- Vendor SDK when available (R85) — only fall back to raw `requests` if no SDK.
- `secrets.token_urlsafe` for idempotency keys — never `random` (R20).
- Specific `except` clauses — never bare `except:` (R17).
- Named `logger` — never `print()` (R18).
- For Stripe: `idempotency_key=` on every `*.create()` call (R81); return-URL
  allowlist check (R69); webhook signature verification (R70 — see
  `webhook.py.md`).
