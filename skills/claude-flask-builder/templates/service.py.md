# Template — Service

Path: `app/domain/service/<Name>Service.py`. Add to `app/domain/service/__init__.py` barrel.

```python
"""<Name>Service — business logic for <feature>. Static methods only."""
import logging
from decimal import Decimal
from app.configuration.Database import db
from app.utils.Validation import Validation, CustomValidationException
from app.utils.Auth import Auth
from app.utils import Constants
from app.domain.validation.<Name>Validation import Create<Name>Validation
from app.domain.dto.<Name>Schema import <Name>Schema
from app.model.<Name> import <Name>

logger = logging.getLogger(__name__)


class <Name>Service:
    """Business logic for the <feature> domain."""

    @staticmethod
    def create(request, current_user):
        """Create a <Name>.
        :raises CustomValidationException: validation failure or duplicate
        """
        data = Validation.validate(request, Create<Name>Validation())

        # Pre-join related entities here (R27 — no DB in @post_dump)
        # ...

        # Single transaction for multi-model writes (R25)
        with db.session.begin_nested():
            obj = <Name>.create(data, commit=False)
            # additional inserts...
        db.session.commit()

        return <Name>Schema().dump(obj)
```

Checklist: @staticmethod (R43), no Flask request parsing beyond Validation.validate (R24), explicit exception types (R17), service owns multi-model transaction (R25), `logger` named (R18).
