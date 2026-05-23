# Template — DTO schema (response serializer)

Path: `app/domain/dto/<Name>Schema.py`. Add to barrel.

```python
"""Response serialization for <feature>."""
from marshmallow import Schema, fields, post_dump


class <Name>Schema(Schema):
    """<Name> response shape."""

    id = fields.Int()
    name = fields.Str()
    amount = fields.Decimal(as_string=False)
    created_at = fields.DateTime()
    deleted_at = fields.DateTime(allow_none=True)

    @post_dump
    def derive(self, data, **kwargs):
        """Enrichment hook — purely computational, NO DB queries (R27).
        Pre-join in the service if relations needed.
        """
        if data == {}:
            return None
        # safe-guarded derived fields only
        return data
```

Checklist: R27 (no DB in post_dump), guard `if data == {}` (P4-5), pre-join in service.
