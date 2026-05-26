# Template — User model

Path: `app/model/User.py`. Add to `app/model/__init__.py` barrel.

This is the canonical shape of the `User` model. It is more specific than
the generic `templates/model.py.md` because every Flask app in this
architecture has the same auth surface and the same column set is required
by `auth-controller.py.md`, `patterns.md §7` (Auth), and the reviewer
(R47 soft-delete, R60 refresh_jti rotation, R65 ptoken, R66 OTP).

Covers rules: R13 (singular PascalCase, plural snake_case table), R40
(indexes on all WHERE columns), R43 (@staticmethod), R45/R19 (no Float —
N/A for User but the pattern stays), R46 (tz-aware timestamps), R47
(explicit `deleted_at`), R61 (explicit field assignment, no
`__dict__.update`), R65 (ptoken column + TTL), R66 (otp column +
attempts), R60 (refresh_jti).

```python
"""User — single source of truth for the users table.

Auth columns:
- `_password` is the bcrypt hash; the `password` hybrid_property hides it
  from default queries and exposes a setter that calls `bcrypt.hashpw`.
- `ptoken` / `ptoken_expires_at` back the password-reset flow (R65).
- `otp` / `otp_created_at` / `otp_attempts` back the OTP flow (R66).
- `refresh_jti` rotates on each refresh; lookups by jti gate refresh-token
  reuse (R60).
"""
import bcrypt
from sqlalchemy import (
    Column, Integer, Text, String, DateTime, Index,
)
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.sql.functions import func

from app.configuration.Database import db
from app.utils import Constants


class User(db.Model):
    """User row."""

    __tablename__ = "users"  # plural snake_case (R13)

    id = Column(Integer, primary_key=True)

    # WHY String(255): RFC 5321 max local-part + domain. lowercase on the
    # way in (Validation layer canonicalizes); UNIQUE constraint enforces
    # no dup accounts.
    email = Column(String(255), nullable=False, unique=True, index=True)
    # WHY String(20): E.164 max length is 15 digits + '+'; cushion to 20.
    # Validation schema MUST canonicalize to E.164 before insert.
    phone = Column(String(20), nullable=False, unique=True, index=True)

    # WHY underscore-prefixed: hides bcrypt hash from default serialization
    # — DTOs never see it (R26 separation; PII / R67 redaction).
    _password = Column("password", Text, nullable=False)

    # WHY String(20) for role: enum string values are short; index for
    # admin-route filtering (R72 will compare against this).
    role = Column(String(20), nullable=False, default=Constants.Role.CUSTOMER.value, index=True)

    # ---- Password reset (R65) ----
    # WHY String(64): secrets.token_urlsafe(32) returns ~43 chars URL-safe;
    # 64 leaves headroom for future bumps to token_urlsafe(48).
    ptoken = Column(String(64), nullable=True, unique=True, index=True)
    ptoken_expires_at = Column(DateTime(timezone=True), nullable=True)

    # ---- OTP (R66) ----
    # WHY String(6): 6-digit numeric OTP (sent as SMS). If you switch to
    # alphanumeric, bump the column AND the generator in AccountService.
    otp = Column(String(6), nullable=True)
    otp_created_at = Column(DateTime(timezone=True), nullable=True)
    otp_attempts = Column(Integer, nullable=False, default=0)

    # ---- Push notifications ----
    device_token = Column(Text, nullable=True)

    # ---- JWT refresh rotation (R60) ----
    # WHY String(32): secrets.token_urlsafe(16) returns ~22 chars; column
    # has headroom so swapping the generator to token_urlsafe(24) (~32
    # chars) does not require a migration.
    refresh_jti = Column(String(32), nullable=True, unique=True, index=True)

    # ---- Soft delete (R47) ----
    deleted_at = Column(DateTime(timezone=True), nullable=True, index=True)

    # ---- Audit timestamps (R46) ----
    created_at = Column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at = Column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )

    __table_args__ = (
        Index("ix_users_role_active", "role", "deleted_at"),
    )

    # ---- Password hybrid (write hashes, read raises) ----
    @hybrid_property
    def password(self) -> str:
        """Reading raw password is forbidden — use `check_password` instead."""
        raise AttributeError("password is write-only; use check_password()")

    @password.setter
    def password(self, plaintext: str) -> None:
        """Hash and store. WHY bcrypt.gensalt(): cost factor 12 default."""
        self._password = bcrypt.hashpw(plaintext.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

    def check_password(self, plaintext: str) -> bool:
        """Constant-time compare via bcrypt.checkpw."""
        return bcrypt.checkpw(plaintext.encode("utf-8"), self._password.encode("utf-8"))

    # ---- Row-instance methods (R43: take self) ----
    def soft_delete(self) -> None:
        """Mark soft-deleted via explicit column (R47 — never email-mangling)."""
        from datetime import datetime, timezone
        self.deleted_at = datetime.now(timezone.utc)

    def to_dict(self) -> dict:
        """Default safe serialization. Excludes _password / ptoken / otp / jti."""
        return {
            "id": self.id, "email": self.email, "phone": self.phone,
            "role": self.role,
            "created_at": self.created_at, "updated_at": self.updated_at,
        }

    # ---- Repository-style methods (R43: @staticmethod) ----
    @staticmethod
    def create(data: dict, commit: bool = True) -> "User":
        """Explicit field assignment (R61) — never `__dict__.update`."""
        user = User(
            email=data["email"].lower(),
            phone=data["phone"],
            role=data.get("role", Constants.Role.CUSTOMER.value),
        )
        user.password = data["password"]  # triggers bcrypt hash via setter
        db.session.add(user)
        if commit:
            db.session.commit()
        return user

    @staticmethod
    def get_by_id(uid: int) -> "User | None":
        """Active (non-deleted) user by id."""
        return db.session.query(User).filter_by(id=uid, deleted_at=None).first()

    @staticmethod
    def get_by_email(email: str) -> "User | None":
        """Active user by canonical lowercased email."""
        return db.session.query(User).filter_by(email=email.lower(), deleted_at=None).first()

    @staticmethod
    def get_by_phone(phone: str) -> "User | None":
        """Active user by phone (caller already normalized to E.164).

        WHY no normalization here: Validation layer owns canonicalization
        (R26). Single source of truth.
        """
        return db.session.query(User).filter_by(phone=phone, deleted_at=None).first()

    @staticmethod
    def get_by_ptoken(ptoken: str) -> "User | None":
        """Active user by password-reset token. Caller checks TTL + single-use (R65)."""
        return db.session.query(User).filter_by(ptoken=ptoken, deleted_at=None).first()

    @staticmethod
    def get_by_refresh_jti(jti: str) -> "User | None":
        """Active user by current refresh-token jti (R60 rotation gate)."""
        return db.session.query(User).filter_by(refresh_jti=jti, deleted_at=None).first()
```

## Checklist (reviewer enforces)
- `_password` is private + hybrid_property; reading `user.password` raises.
- `bcrypt` for hashing + `bcrypt.checkpw` for compare (constant-time).
- `email.lower()` on every write + read path (canonical).
- `phone` NOT normalized in the model — Validation layer is the single
  normalizer (R26).
- `ptoken` column is `String(64)` with `unique=True, index=True` (R40/R65).
- `otp_attempts` is non-nullable with default 0 (R66 counter contract).
- `refresh_jti` is `String(32)` with `unique=True, index=True` (R60).
- `deleted_at` exists; all `get_by_*` lookups filter `deleted_at=None` (R47).
- Repository methods are `@staticmethod`; row-instance methods take `self` (R43).
- No `__dict__.update` (R61).
- All `DateTime` columns are `timezone=True` (R46).
