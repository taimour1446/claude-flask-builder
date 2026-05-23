# Template — Model

Path: `app/model/<Name>.py`. Add to barrel.

```python
"""<Name> model — single source of truth for the <name>s table."""
from decimal import Decimal
from sqlalchemy import Column, Integer, Text, Boolean, Numeric, DateTime, ForeignKey, Index
from sqlalchemy.sql.functions import func
from app.configuration.Database import db


class <Name>(db.Model):
    """<Name> row."""
    __tablename__ = '<name>s'  # plural snake_case (R13)

    id = Column(Integer, primary_key=True)
    name = Column(Text, nullable=False, index=True)            # R40 — indexed
    amount = Column(Numeric(10, 2), nullable=False, default=0) # R45 — Numeric not Float
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False, index=True)
    deleted_at = Column(DateTime(timezone=True), nullable=True, index=True)  # R47 — explicit soft delete
    created_at = Column(DateTime(timezone=True), nullable=False, server_default=func.now())  # R46
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    __table_args__ = (
        Index('ix_<name>s_user_active', 'user_id', 'deleted_at'),
    )

    def to_dict(self) -> dict:
        """Return a dict copy of all columns."""
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

    @staticmethod
    def create(data: dict, commit: bool = True):
        """Create a row. Pass commit=False when called inside a service transaction."""
        # Explicit field assignment (R61) — no model.__dict__.update(data)
        obj = <Name>(
            name=data['name'],
            amount=Decimal(str(data['amount'])),
            user_id=data['user_id'],
        )
        db.session.add(obj)
        if commit:
            db.session.commit()
        return obj

    @staticmethod
    def get_by_id(id: int):
        """Fetch an active (non-deleted) <Name> by id."""
        return db.session.query(<Name>).filter_by(id=id, deleted_at=None).first()

    @staticmethod
    def exists_by_name(name: str) -> bool:
        """Uniqueness check used by validation."""
        return db.session.query(<Name>.id).filter_by(name=name, deleted_at=None).first() is not None
```

Checklist: R13 (plural snake_case), R40 (indexes), R43 (@staticmethod), R45 (Numeric), R46 (tz-aware), R47 (deleted_at), R61 (no __dict__.update), R44 (no `default=callable()` — pass the callable, not its result).
