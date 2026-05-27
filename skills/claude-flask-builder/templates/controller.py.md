# Template — Controller (flask-smorest Blueprint)

Path: `app/api/<Name>Controller.py`. Add to `app/api/__init__.py` barrel.
Register via `api.register_blueprint(<name>_blueprint(), ...)` in
`Configuration.register_api` (NOT plain `app.register_blueprint`).

The Blueprint MUST be `flask_smorest.Blueprint` (NOT `flask.Blueprint`) so
that `@blp.arguments(<Validation>)` + `@blp.response(<status>, <DTO>)`
auto-generate the OpenAPI spec from the existing Marshmallow schemas
(R26/R27). The controller body stays a thin pass-through to the service —
R23 still applies.

```python
"""<Name>Controller — flask-smorest blueprint for <feature>."""
import logging

from flask import request
from flask_smorest import Blueprint

from app.domain.dto.<Name>Schema import <Name>Schema
from app.domain.service.<Name>Service import <Name>Service
from app.domain.validation.<Name>Validation import (
    Create<Name>Validation, Update<Name>Validation,
)
from app.utils.Auth import token_required, role_required
from app.utils.BaseResponse import BaseResponse
from app.utils.Validation import CustomValidationException

logger = logging.getLogger(__name__)


def <name>_blueprint() -> Blueprint:
    """Build and return the <name> blueprint.

    WHY flask_smorest.Blueprint: subclass of flask.Blueprint that
    understands `@blp.arguments` + `@blp.response`. Standard Flask
    routes still work (`@bp.route`) — the decorators are additive.
    """
    bp = Blueprint(
        "<name>", __name__,
        url_prefix="/<resource>",
        description="<Name> endpoints",  # → OpenAPI tag description
    )

    @bp.route("", methods=["POST"])
    @bp.arguments(Create<Name>Validation)   # → request body in OpenAPI spec
    @bp.response(200, <Name>Schema)         # → response shape in OpenAPI spec
    @bp.doc(security=[{"BearerAuth": []}])  # → padlock icon in Swagger UI
    @token_required
    @role_required(["Admin", "SubAdmin"])
    def create_<name>(payload: dict, current_user) -> BaseResponse:
        """POST /<resource> — create a <Name>."""
        try:
            data = <Name>Service.create(request, current_user)
            return BaseResponse.respond(data)
        except CustomValidationException as e:
            return BaseResponse.respondError(message=str(e))
        except Exception:
            logger.exception("create_<name>_failed")
            return BaseResponse.respondError(message="Internal error")

    return bp
```

Notes on decorator ordering (flask-smorest contract):
1. `@bp.route(...)` first (the path).
2. `@bp.arguments(<Schema>)` — passes validated dict as the FIRST positional
   arg to the handler (here named `payload`). The skill still runs
   Validation.validate inside the service to keep R26 single-source-of-truth
   — `@bp.arguments` is for OpenAPI emission, not double-validation.
3. `@bp.response(<status>, <Schema>)` — emits the response shape.
4. `@bp.doc(security=[...])` — toggles auth requirement IN the OpenAPI spec.
   Public routes use `@bp.doc(security=[])` to clear the global Bearer
   requirement.
5. `@token_required` + `@role_required` LAST (closest to the function) so
   they run first at request time.

Checklist (reviewer enforces):
- R23 (no business logic in controller).
- R72 (role asserted).
- R17 (no bare except).
- Controller uses `flask_smorest.Blueprint` (NOT plain `flask.Blueprint`).
- Every method has BOTH `@bp.arguments` (if it accepts a body) AND
  `@bp.response` — otherwise the OpenAPI spec is incomplete.
- Protected routes carry `@bp.doc(security=[{"BearerAuth": []}])`;
  public routes carry `@bp.doc(security=[])`.
- Comments on factory + each handler.
- Blueprint name is snake_case (R11).
