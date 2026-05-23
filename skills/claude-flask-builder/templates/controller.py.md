# Template — Controller

Path: `app/api/<Name>Controller.py`. Add to `app/api/__init__.py` barrel + register in `application.py`.

```python
"""<Name>Controller — blueprint for <feature>. Thin pass-through to <Name>Service."""
import logging
from flask import Blueprint, request
from app.domain.service import <Name>Service
from app.utils.Auth import token_required, role_required
from app.utils.BaseResponse import BaseResponse
from app.utils.Validation import CustomValidationException

logger = logging.getLogger(__name__)


def <name>_blueprint() -> Blueprint:
    """Build and return the <name> blueprint."""
    bp = Blueprint('<name>', __name__)

    @bp.route('/<resource>', methods=['POST'])
    @token_required
    @role_required(['Admin', 'SubAdmin'])
    def create_<name>(current_user) -> BaseResponse:
        """POST /<resource> — create a <Name>."""
        try:
            data = <Name>Service.create(request, current_user)
            return BaseResponse.respond(data)
        except CustomValidationException as e:
            return BaseResponse.respondError(message=str(e))
        except Exception:
            # logger.exception captures stack; respondError stays generic in prod
            logger.exception('create_<name>_failed')
            return BaseResponse.respondError(message='Internal error')

    return bp
```

Checklist: R23 (no business logic), R72 (role asserted), R17 (no bare except), comments on factory + each handler, blueprint name is snake_case.
