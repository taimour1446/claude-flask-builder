# Template — AccountValidation (6 auth schemas)

Path: `app/domain/validation/AccountValidation.py`. Add to
`app/domain/validation/__init__.py` barrel — explicit re-export of every
class.

This is the canonical shape for the six auth-flow validation schemas
referenced in `templates/auth-controller.py.md`. All six live in one file
because they share the same `User` lookups + the same field vocabulary
(email/phone/password/otp/ptoken).

Covers rules: R26 (Validation owns request parsing), R28
(`@validates_schema` raises `CustomValidationException` not
`ValidationError`), R62 (`unknown=EXCLUDE`), R65 (password length 12-72
for bcrypt), R66 (OTP is 6 digits).

```python
"""AccountValidation — six request schemas for the /auth/* surface."""
import re

from marshmallow import Schema, fields, validate, validates_schema, EXCLUDE

from app.model.User import User
from app.utils import Constants
from app.utils.Validation import CustomValidationException


# WHY centralized regex: phone canonicalization is the Validation layer's
# job (R26). Anywhere a phone field is read, normalize it HERE so the
# service receives an E.164 string.
_E164_RE = re.compile(r"^\+[1-9]\d{7,14}$")


def _normalize_email(value: str) -> str:
    return value.strip().lower()


def _normalize_phone(value: str) -> str:
    """Strip whitespace; require E.164 (`+` + country code + digits, 8-15 total)."""
    stripped = value.strip().replace(" ", "").replace("-", "")
    if not _E164_RE.match(stripped):
        raise CustomValidationException("Phone must be in E.164 format (e.g. +14155552671)")
    return stripped


# ---- Field re-usables ----
_email_field = fields.String(
    required=True,
    validate=[validate.Length(min=5, max=255), validate.Email()],
    error_messages={"required": "Email is required"},
)
_phone_field = fields.String(
    required=True,
    validate=validate.Length(min=8, max=20),
    error_messages={"required": "Phone is required"},
)
# WHY min=12, max=72: NIST SP 800-63B recommends ≥12 chars; bcrypt
# truncates at 72 bytes — accepting >72 would let two different inputs
# hash identically (see `user-model.py.md` password setter docstring).
_password_field = fields.String(
    required=True,
    validate=validate.Length(min=12, max=72),
    error_messages={
        "required": "Password is required",
    },
)
_otp_field = fields.String(
    required=True,
    validate=validate.Regexp(r"^\d{6}$", error="OTP must be exactly 6 digits"),
    error_messages={"required": "OTP is required"},
)
_ptoken_field = fields.String(
    required=True,
    validate=validate.Length(equal=43),  # secrets.token_urlsafe(32) → 43 chars
    error_messages={"required": "Token is required"},
)


# ---- Schemas ----

class SignupValidation(Schema):
    """POST /auth/signup — create a new account."""

    class Meta:
        unknown = EXCLUDE  # R62

    email = _email_field
    phone = _phone_field
    password = _password_field

    @validates_schema
    def _normalize_and_check(self, data: dict, **kwargs) -> None:
        """Normalize email + phone; reject if email/phone already taken (R28).

        WHY raise generic REGISTRATION_FAILED for duplicates: prevents
        account enumeration. The specific reason is logged server-side
        only. Trade-off: legitimate users with a forgotten account get a
        less-helpful message; spec'd as accepted.
        """
        data["email"] = _normalize_email(data["email"])
        data["phone"] = _normalize_phone(data["phone"])
        if User.get_by_email(data["email"]) or User.get_by_phone(data["phone"]):
            raise CustomValidationException(Constants.REGISTRATION_FAILED)


class LoginValidation(Schema):
    """POST /auth/login — issue tokens."""

    class Meta:
        unknown = EXCLUDE

    email = _email_field
    password = fields.String(
        required=True,
        validate=validate.Length(min=1, max=72),  # accept any length on login attempt
        error_messages={"required": "Password is required"},
    )

    @validates_schema
    def _normalize(self, data: dict, **kwargs) -> None:
        data["email"] = _normalize_email(data["email"])


class ForgotPasswordValidation(Schema):
    """POST /auth/forgot-password — request a reset token."""

    class Meta:
        unknown = EXCLUDE

    email = _email_field

    @validates_schema
    def _normalize(self, data: dict, **kwargs) -> None:
        data["email"] = _normalize_email(data["email"])


class ResetPasswordValidation(Schema):
    """POST /auth/reset-password — consume reset token + set new password."""

    class Meta:
        unknown = EXCLUDE

    ptoken = _ptoken_field
    password = _password_field


class SendOTPValidation(Schema):
    """POST /auth/send-otp — issue 6-digit OTP via SMS."""

    class Meta:
        unknown = EXCLUDE

    phone = _phone_field

    @validates_schema
    def _normalize(self, data: dict, **kwargs) -> None:
        data["phone"] = _normalize_phone(data["phone"])


class VerifyOTPValidation(Schema):
    """POST /auth/verify-otp — exchange OTP for tokens."""

    class Meta:
        unknown = EXCLUDE

    phone = _phone_field
    otp = _otp_field

    @validates_schema
    def _normalize(self, data: dict, **kwargs) -> None:
        data["phone"] = _normalize_phone(data["phone"])
```

## Checklist (reviewer enforces)
- Every Schema has `class Meta: unknown = EXCLUDE` (R62).
- `@validates_schema` methods raise `CustomValidationException` (R28).
- Password length is bounded `12..72` to avoid bcrypt truncation
  ambiguity AND meet NIST 800-63B (R65 cross-tie).
- OTP is exactly 6 digits (R66 — matches the 6-digit shape generated by
  `secrets.randbelow(1_000_000)`).
- Phone is normalized to E.164 IN the validator — model layer does not
  normalize (single source of truth per R26; see `user-model.py.md`
  `get_by_phone` docstring).
- Email is lowercased IN the validator — same reasoning.
- Signup uniqueness check returns generic `REGISTRATION_FAILED`
  (enum-resistance).
