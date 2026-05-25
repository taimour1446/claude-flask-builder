# Template — Validation schema

Path: `app/domain/validation/<Name>Validation.py`. Add to barrel.

```python
"""Request-validation schemas for <feature>."""
from marshmallow import Schema, fields, validates_schema, EXCLUDE, validate
from app.utils.Validation import CustomValidationException
from app.utils import Constants
from app.model.<Name> import <Name>


class Create<Name>Validation(Schema):
    """Validate Create<Name> request payload."""

    class Meta:
        unknown = EXCLUDE  # R62 — never INCLUDE (mass-assignment risk)

    name = fields.String(
        required=True,
        validate=validate.Length(min=1, max=255),
        error_messages={'required': 'Name is required'},
    )
    amount = fields.Decimal(
        required=True,
        as_string=False,
        error_messages={'required': 'Amount is required'},
    )

    @validates_schema
    def custom(self, data: dict, **kwargs) -> None:
        """Cross-field and uniqueness checks. Raise CustomValidationException."""
        if <Name>.exists_by_name(data['name']):
            raise CustomValidationException(Constants.NameTaken)
```

Checklist: `unknown = EXCLUDE` (R62), Decimal for money (R19), error_messages set (P4-12), @validates_schema raises CustomValidationException not ValidationError (R28).
