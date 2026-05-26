# Template — AccountSchema (DTO for User responses)

Path: `app/domain/dto/AccountSchema.py`. Add to `app/domain/dto/__init__.py`
barrel.

This is the only DTO allowed to serialize a `User` row. It MUST exclude
sensitive fields: `_password`, `ptoken`, `ptoken_expires_at`, `otp`,
`otp_created_at`, `otp_attempts`, `refresh_jti`, `device_token` (per R26
serialization layer + R67 PII-redaction posture).

Covers rules: R26 (DTO layer for serialization, never `Schema().load(...)`
on responses), R27 (`@post_dump` MUST NOT query DB — pre-join in service),
R67 (do not leak PII / secrets to clients).

```python
"""AccountSchema — User → response DTO. Excludes sensitive fields by design."""
from marshmallow import Schema, fields, post_dump


class AccountSchema(Schema):
    """Public-safe view of a User row.

    Whitelisted fields ONLY — every field listed below is explicitly safe
    to return to a client. Adding a new column to `User` does NOT
    automatically expose it. The reviewer FAILs any change that adds
    `_password` / `ptoken*` / `otp*` / `refresh_jti` / `device_token` to
    this schema.
    """

    id = fields.Int()
    email = fields.Str()
    phone = fields.Str()
    role = fields.Str()
    created_at = fields.DateTime()
    updated_at = fields.DateTime()

    @post_dump
    def _strip_empty(self, data: dict, **kwargs) -> dict | None:
        """R27: ZERO DB queries here. Drop empty dicts so list serializers
        can return `null` for missing rows. Pre-join any extra data in the
        service layer (R27)."""
        if not data:
            return None
        return data
```

## Forbidden fields (do not add — reviewer FAILs)
- `_password` / `password` — bcrypt hash; never exposed.
- `ptoken` / `ptoken_expires_at` — password-reset token; would let
  anyone with the response reset the user's password.
- `otp` / `otp_created_at` / `otp_attempts` — bypass MFA.
- `refresh_jti` — letting the client see this defeats R60 rotation.
- `device_token` — used to send push; not the client's business in this
  endpoint.

If you need to surface a derived field (e.g. `display_name`), compose it
in the service via `Helper`/preprocessing and pass the value through the
schema as an explicit `fields.Method` — do NOT add a DB query in
`@post_dump` (R27).

## Checklist (reviewer enforces)
- Whitelist-only — no `Meta.fields = "__all__"` or `Meta.exclude` games.
- Forbidden fields above are absent.
- `@post_dump` does not import `db` or call any `.query.*`.
- File header docstring + class docstring present (R30/R31).
