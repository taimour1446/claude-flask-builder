# Template — Application factory

Path: `app/application.py`. Single create_application() per project.

```python
"""Application factory. Order of init matters."""
from flask import Flask
from app.configuration.Configuration import (
    configure_logging, configure_settings, configure_application,
    configure_database, configure_inject, configure_mail,
    register_error_handlers, register_blueprints, register_scheduler_jobs,
)
from app.utils.RequestInterceptor import RequestInterceptor


def create_application() -> Flask:
    """Build and return the Flask app — production-ready."""
    app = Flask(__name__)
    configure_logging(app)        # logger first so subsequent logs work
    configure_settings(app)       # Settings.validate() — fail fast on missing env
    configure_application(app)    # CORS, JSON config, app-level
    configure_database(app)       # db.init_app — NO db.create_all (Alembic owns schema)
    configure_inject(app)         # DI binder (empty by default)
    configure_mail(app)
    register_error_handlers(app)  # global @errorhandler set (R90)
    register_blueprints(app)      # all blueprints under /api/v1
    RequestInterceptor.intercept(app)  # X-Request-ID + redacted logging
    register_scheduler_jobs(app)
    return app
```
