# Template — Alembic migration

Path: `migrations/versions/<rev>_<descriptive_slug>.py` (always use slug — R50).

```python
"""<descriptive slug>

Revision ID: <rev>
Revises: <down_rev>
Create Date: <date>
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = '<rev>'
down_revision = '<down_rev>'
branch_labels = None
depends_on = None


def upgrade():
    """Apply schema change."""
    op.create_table(
        '<name>s',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.Text(), nullable=False),
        sa.Column('amount', sa.Numeric(10, 2), nullable=False, server_default='0'),
        sa.Column('user_id', sa.Integer(), sa.ForeignKey('users.id'), nullable=False),
        sa.Column('deleted_at', sa.DateTime(timezone=True), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=False),
    )
    op.create_index('ix_<name>s_name', '<name>s', ['name'])
    op.create_index('ix_<name>s_user_active', '<name>s', ['user_id', 'deleted_at'])


def downgrade():
    """Reverse of upgrade — R49: NEVER `pass`."""
    op.drop_index('ix_<name>s_user_active', table_name='<name>s')
    op.drop_index('ix_<name>s_name', table_name='<name>s')
    op.drop_table('<name>s')
```

Zero-downtime add-NOT-NULL: do it in 3 migrations (add nullable → backfill → enforce). P19-8.
