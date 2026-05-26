# Template — Auth controller (login, signup, reset password, OTP)

Path: `app/api/AccountController.py`. Paired service `AccountService` in
`app/domain/service/AccountService.py`.

Covers rules: R60 (JWT exp claim), R65 (password reset token ≥32 bytes +
≤15-min TTL + single-use), R66 (OTP TTL ≤5 min + max-attempts), R72 (role
assertion in service), all standard 4-layer rules.

```python
"""AccountController — authentication surface (login/signup/reset/otp)."""
import logging

from flask import Blueprint, request

from app.domain.service.AccountService import AccountService
from app.utils.Auth import token_required
from app.utils.BaseResponse import BaseResponse
from app.utils.Validation import CustomValidationException

logger = logging.getLogger(__name__)


def account_blueprint() -> Blueprint:
    bp = Blueprint("account", __name__)

    @bp.route("/auth/signup", methods=["POST"])
    def signup() -> BaseResponse:
        """Create a customer account. Returns tokens on success."""
        try:
            data = AccountService.signup(request)
            return BaseResponse.respond(data, message="Signup successful")
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc))
        except Exception:
            logger.exception("signup_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/login", methods=["POST"])
    def login() -> BaseResponse:
        """Validate credentials and issue access + refresh tokens (R60)."""
        try:
            data = AccountService.login(request)
            return BaseResponse.respond(data)
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc))
        except Exception:
            logger.exception("login_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/forgot-password", methods=["POST"])
    def forgot_password() -> BaseResponse:
        """Issue a password-reset token (R65: 32-byte secret, 15-min TTL, single-use)."""
        try:
            AccountService.forgot_password(request)
            # WHY always 200: never confirm whether the email exists (enum-resistance).
            return BaseResponse.respond(None, message="If the email exists, a reset link was sent")
        except Exception:
            logger.exception("forgot_password_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/reset-password", methods=["POST"])
    def reset_password() -> BaseResponse:
        """Consume reset token and set new password (R65: single-use, TTL check)."""
        try:
            AccountService.reset_password(request)
            return BaseResponse.respond(None, message="Password updated")
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc))
        except Exception:
            logger.exception("reset_password_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/send-otp", methods=["POST"])
    def send_otp() -> BaseResponse:
        """Send an OTP via SMS / email (R66: 5-min TTL)."""
        try:
            AccountService.send_otp(request)
            return BaseResponse.respond(None, message="OTP sent")
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc))
        except Exception:
            logger.exception("send_otp_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/verify-otp", methods=["POST"])
    def verify_otp() -> BaseResponse:
        """Verify an OTP (R66: max-attempts rate limit, attempts counter)."""
        try:
            data = AccountService.verify_otp(request)
            return BaseResponse.respond(data)
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc))
        except Exception:
            logger.exception("verify_otp_failed")
            return BaseResponse.respondError(message="Internal error")

    @bp.route("/auth/refresh", methods=["POST"])
    def refresh() -> BaseResponse:
        """Exchange a valid refresh token for a new access token (R60: jti rotation)."""
        try:
            data = AccountService.refresh(request)
            return BaseResponse.respond(data)
        except CustomValidationException as exc:
            return BaseResponse.respondError(message=str(exc), auth_error=1)
        except Exception:
            logger.exception("refresh_failed")
            return BaseResponse.respondError(message="Internal error", auth_error=1)

    @bp.route("/auth/me", methods=["GET"])
    @token_required
    def me(current_user) -> BaseResponse:
        """Return the calling user's profile."""
        return BaseResponse.respond(AccountService.serialize(current_user))

    return bp
```

## Companion service shape (key methods)

```python
"""AccountService — owns auth business logic. Static methods only (R43)."""
import datetime as dt
import logging
import secrets

import bcrypt

from app.configuration.Database import db
from app.domain.dto.AccountSchema import AccountSchema
from app.domain.validation.AccountValidation import (
    SignupValidation, LoginValidation, ForgotPasswordValidation,
    ResetPasswordValidation, SendOTPValidation, VerifyOTPValidation,
)
from app.model.User import User
from app.utils import Constants
from app.utils.Auth import Auth
from app.utils.Validation import Validation, CustomValidationException

logger = logging.getLogger(__name__)


class AccountService:

    @staticmethod
    def forgot_password(request) -> None:
        """Issue a reset token. ALWAYS 200 — never confirm email existence."""
        data = Validation.validate(request, ForgotPasswordValidation())
        user = User.get_by_email(data["email"])
        if user is None:
            return  # silent — enum-resistance
        # R65: 32-byte cryptographic token, 15-min TTL, single-use
        user.ptoken = secrets.token_urlsafe(32)
        user.ptoken_expires_at = dt.datetime.now(dt.timezone.utc) + dt.timedelta(minutes=15)
        db.session.commit()
        # ... mail it via Flask-Mail (template in templates/emails/)

    @staticmethod
    def reset_password(request) -> None:
        """Single-use: clear ptoken on success."""
        data = Validation.validate(request, ResetPasswordValidation())
        user = User.get_by_ptoken(data["ptoken"])
        if user is None:
            raise CustomValidationException(Constants.INVALID_TOKEN)
        now = dt.datetime.now(dt.timezone.utc)
        if user.ptoken_expires_at is None or user.ptoken_expires_at < now:
            raise CustomValidationException(Constants.TOKEN_EXPIRED)
        user._password = bcrypt.hashpw(data["password"].encode(), bcrypt.gensalt()).decode()
        user.ptoken = None  # R65: single-use — invalidate immediately
        user.ptoken_expires_at = None
        # WHY rotate refresh_jti: force re-login on all other devices after a password reset
        user.refresh_jti = secrets.token_urlsafe(16)
        db.session.commit()

    @staticmethod
    def send_otp(request) -> None:
        """6-digit OTP. R66: 5-min TTL, attempts reset."""
        data = Validation.validate(request, SendOTPValidation())
        user = User.get_by_phone(data["phone"])
        if user is None:
            return  # silent
        # WHY secrets.randbelow not random: R20 — security-critical PRNG
        user.otp = f"{secrets.randbelow(1_000_000):06d}"
        user.otp_created_at = dt.datetime.now(dt.timezone.utc)
        user.otp_attempts = 0
        db.session.commit()
        # ... send via TwilioClient.send_sms(...)

    @staticmethod
    def verify_otp(request) -> dict:
        """R66: max 5 attempts/hour, 5-min TTL. Issues tokens on success."""
        data = Validation.validate(request, VerifyOTPValidation())
        user = User.get_by_phone(data["phone"])
        if user is None or user.otp is None:
            raise CustomValidationException(Constants.INVALID_OTP)
        now = dt.datetime.now(dt.timezone.utc)
        # R66: TTL
        if (now - user.otp_created_at).total_seconds() > 300:
            raise CustomValidationException(Constants.OTP_EXPIRED)
        # R66: max-attempts
        if user.otp_attempts >= 5:
            raise CustomValidationException(Constants.TOO_MANY_OTP_ATTEMPTS)
        if not secrets.compare_digest(user.otp, data["otp"]):
            user.otp_attempts += 1
            db.session.commit()
            raise CustomValidationException(Constants.INVALID_OTP)
        # Success: consume + rotate refresh jti
        user.otp = None
        user.otp_created_at = None
        user.otp_attempts = 0
        user.refresh_jti = secrets.token_urlsafe(16)
        db.session.commit()
        return Auth.generate_tokens(user)

    # ... signup / login / refresh follow the same shape ...
```

## Checklist (reviewer enforces)
- Every controller method wraps service in try/except CustomValidationException + Exception (R23).
- Service uses `secrets.*` for ptoken / OTP / jti — never `random` (R20).
- ptoken: ≥32 bytes via `secrets.token_urlsafe(32)`, 15-min TTL, single-use (R65).
- OTP: ≤5-min TTL, max-attempts counter, `secrets.compare_digest` for constant-time compare (R66).
- `forgot_password` and `send_otp` return 200 even when user not found (enum-resistance).
- `Auth.generate_tokens(user)` includes `exp` claim and rotates `refresh_jti` (R60).
- `@token_required` on `me` / private routes (R60).
