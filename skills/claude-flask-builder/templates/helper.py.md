# Template — Helper utilities

Path: `app/utils/Helper.py`. Single source for pagination, secrets-based
token generation, and dict merging.

Covers rules: R20 (`secrets` only — NEVER `random` for security-critical
PRNG), R65 (ptoken ≥32 raw bytes via `secrets.token_urlsafe(32)`).

```python
"""Helper — pagination + secrets-based token generators + small utilities.

WHY locked implementations:
- generate_otp / generate_ptoken use `secrets` not `random` (R20). Random
  is predictable; `secrets` uses os.urandom under the hood.
- ptoken length is fixed at 32 raw bytes (= 43 URL-safe chars). User
  model's `ptoken` column is String(64) — fits with headroom for future
  bumps (R65).
- pagination shape is locked at `{"pagination": {...}, "list": [...]}`
  because every list endpoint's DTO consumer relies on it.
"""
import math
import secrets
from typing import Any

from marshmallow import Schema


def generate_otp() -> str:
    """6-digit numeric OTP, zero-padded. Caller stores + verifies (R66).

    WHY secrets.randbelow not random.randint: cryptographic PRNG. random.*
    is seeded from system clock by default — predictable across processes.
    """
    return f"{secrets.randbelow(1_000_000):06d}"


def generate_ptoken() -> str:
    """Password-reset token: 32 raw bytes → 43 URL-safe chars (R65).

    Caller writes the result to User.ptoken + sets ptoken_expires_at
    (TTL ≤15 min). On reset_password success the column is cleared
    (single-use).
    """
    return secrets.token_urlsafe(32)


def generate_jti() -> str:
    """JWT refresh-token jti — 16 raw bytes → 22 URL-safe chars (R60).

    User.refresh_jti column is String(32); 22 chars fits with headroom
    so future swaps to `secrets.token_urlsafe(24)` (~32 chars) do not
    require a migration.
    """
    return secrets.token_urlsafe(16)


def pagination(schema: Schema, items: list[Any], query: dict) -> dict:
    """Offset-based pagination. Returns the locked client envelope.

    :param schema: Marshmallow schema (many=False) used to dump each row.
    :param items: full list-or-cursor from the service layer.
    :param query: dict from `request.args.to_dict()` — reads `page`,
        `limit` (both optional ints with sane defaults).
    :returns: `{"pagination": {"total", "page", "pageCount", "limit"}, "list": [...]}`

    WHY offset (not cursor): the reference codebase shipped offset; clients
    expect it. For tables expected to exceed 10k rows, swap this with a
    cursor-based variant — keep the OUTER shape identical.
    """
    page = max(1, int(query.get("page", 1) or 1))
    limit = max(1, min(int(query.get("limit", 20) or 20), 100))  # cap at 100
    total = len(items)
    page_count = math.ceil(total / limit) if total else 0
    offset = (page - 1) * limit
    sliced = items[offset:offset + limit]
    return {
        "pagination": {
            "total": total,
            "page": page,
            "pageCount": page_count,
            "limit": limit,
        },
        "list": [schema.dump(row) for row in sliced],
    }


def merge_dict(base: dict, override: dict) -> dict:
    """Recursive dict merge. `override` wins on conflict.

    WHY recursive (not `{**base, **override}`): shallow merge loses nested
    keys. e.g. base={'a':{'x':1}}, override={'a':{'y':2}} should give
    {'a':{'x':1,'y':2}}, not {'a':{'y':2}}.
    """
    out = dict(base)
    for k, v in override.items():
        if k in out and isinstance(out[k], dict) and isinstance(v, dict):
            out[k] = merge_dict(out[k], v)
        else:
            out[k] = v
    return out
```

## Checklist (reviewer enforces)
- `generate_otp` uses `secrets.randbelow` — NEVER `random.*` (R20).
- `generate_ptoken` returns `secrets.token_urlsafe(32)` — ≥32 raw bytes,
  matches the `User.ptoken` String(64) column headroom (R65).
- `generate_jti` returns `secrets.token_urlsafe(16)` — matches
  `User.refresh_jti` String(32) headroom (R60).
- `pagination` returns the exact `{"pagination": {...}, "list": [...]}`
  envelope — clients depend on it. The reviewer FAILs any divergence.
- `pagination` caps `limit` at 100 — prevents abusive `?limit=999999`.
- `merge_dict` is recursive — shallow merge would silently lose nested
  keys (subtle bug).
