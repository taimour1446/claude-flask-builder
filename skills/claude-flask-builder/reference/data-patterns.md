# Data Patterns

## SQLAlchemy
- Column types: stdlib `from sqlalchemy import Column, Integer, Text, ...` (NOT `from sqlalchemy.sql.sqltypes`)
- Money: `Numeric(10, 2)` (R45)
- Timestamps: `DateTime(timezone=True)` always (R46)
- JSON: `Column(JSON, server_default='{}'|'[]')` or MutableList for in-place mutation
- Enums: `Column(Enum(MyEnum, name='my_enum'), default=MyEnum.X)` — give explicit name= for Postgres
- FKs: use real `ForeignKey('users.id')` — NOT plain Integer (corrects reference)
- Indexes: every WHERE column gets `index=True` on declaration OR explicit `op.create_index` in migration (R40)

## Transactions
- Models own single-row writes (commit inside).
- Services own multi-model writes via `with db.session.begin_nested():` SAVEPOINT, OR by suppressing commit in models and calling `db.session.commit()` at end of service.
- After `.update(synchronize_session=False)` → `db.session.expire_all()` (R48)

## Pagination
- Helper.pagination(SchemaClass(many=True), data, query) returns `{pagination: {total, page, pageCount, limit}, list: [...]}`
- Offset by default; cursor pagination for endpoints expected to be hot (>10k rows)

## Bulk operations
- Use `bulk_insert_mappings` for >100 rows
- Wrap in single transaction

## Migrations (Alembic)
- alembic.ini: `script_location = migrations` (NOT db/)
- Every migration has working downgrade (R49)
- Descriptive slug: `flask db migrate -m "add idx on order.serial"`
- Indexes: `op.create_index('ix_users_email', 'users', ['email'], unique=True)`
- Zero-downtime for NOT NULL: (1) add nullable + default → (2) backfill data → (3) enforce NOT NULL
- batch_alter_table for SQLite compat
- `from sqlalchemy.dialects import postgresql` for ENUM

## Composite uniques
Required at scaffold time:
- OrderRequest: UNIQUE(order_id, driver_id)
- DriverSlot: UNIQUE(user_id, slot_id, date)
- Fare: UNIQUE(key)
- User: UNIQUE(email), UNIQUE(phone)

### Migration syntax (R41)
```python
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    op.create_table(
        "order_requests",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("order_id", sa.Integer, sa.ForeignKey("orders.id"), nullable=False),
        sa.Column("driver_id", sa.Integer, sa.ForeignKey("users.id"), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True),
                  server_default=sa.func.now(), nullable=False),
        sa.UniqueConstraint("order_id", "driver_id", name="uq_order_request_order_driver"),
        sa.Index("ix_order_request_driver", "driver_id"),
    )

def downgrade() -> None:
    op.drop_table("order_requests")  # R49 — real body, never `pass`
```

If a new model legitimately has NO duplicate-prevention pair, document it in
the migration's docstring:
```python
"""add foo_audit_log table.

No composite UNIQUE: append-only log; duplicates are valid (R41 waived).
"""
```
The reviewer's R41 check passes if EITHER a UniqueConstraint exists OR the
docstring explains why none applies.

## Idempotency
- DB-level UNIQUE + catch IntegrityError → return existing
- Stripe mutations: `idempotency_key=` always (R81)
- Cron jobs: max_instances=1 + distributed lock (Postgres advisory)
