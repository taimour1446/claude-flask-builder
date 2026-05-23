# Architecture — the 4-layer Flask API

```
<project-root>/
├── api/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── application.py          # create_application() factory
│   │   ├── api/                    # CONTROLLER layer (blueprints)
│   │   │   ├── __init__.py         # barrel: from .XController import x_blueprint
│   │   │   └── XController.py      # one per resource
│   │   ├── domain/
│   │   │   ├── service/            # SERVICE layer (business logic, static methods)
│   │   │   │   ├── __init__.py
│   │   │   │   └── XService.py
│   │   │   ├── validation/         # request validation (Marshmallow)
│   │   │   │   ├── __init__.py
│   │   │   │   └── XValidation.py
│   │   │   └── dto/                # response DTO (Marshmallow)
│   │   │       ├── __init__.py
│   │   │       └── XSchema.py
│   │   ├── model/                  # DATA layer (SQLAlchemy)
│   │   │   ├── __init__.py
│   │   │   └── X.py
│   │   ├── configuration/
│   │   │   ├── __init__.py
│   │   │   ├── Configuration.py    # configure_application/database/inject/mail/logging
│   │   │   ├── Database.py         # db = SQLAlchemy()
│   │   │   ├── Mail.py             # mail = Mail()
│   │   │   └── Scheduler.py        # scheduler = APScheduler() (correct spelling)
│   │   ├── utils/
│   │   │   ├── Auth.py             # JWT + token_required + role-required decorators
│   │   │   ├── BaseResponse.py     # respond/respondError envelope
│   │   │   ├── Constants.py        # ALL enums/classes
│   │   │   ├── Helper.py           # pagination, secrets-based generators, mergeDict
│   │   │   ├── Logging.py          # structured logging config
│   │   │   ├── RequestInterceptor.py  # before/after request + X-Request-ID + redaction
│   │   │   ├── Settings.py         # validated env config (single source)
│   │   │   ├── TimezoneHelper.py   # pytz/zoneinfo
│   │   │   └── Validation.py       # universal Validation.validate(request, Schema())
│   │   ├── scheduler_jobs/         # NEW: cron jobs separated from controllers (P6-12)
│   │   ├── integrations/           # external API wrappers (one file per provider)
│   │   ├── templates/
│   │   │   └── emails/             # Jinja2 with base layout
│   │   └── tests/                  # pytest (NEW — reference had none)
│   ├── migrations/
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions/               # empty in scaffold
│   ├── seeds/
│   ├── Pipfile
│   ├── Dockerfile                  # multi-stage, non-root
│   ├── docker-entrypoint.sh        # wait-for-db → alembic → gunicorn
│   ├── alembic.ini                 # script_location=migrations (NOT db/)
│   ├── pyproject.toml              # ruff/black/isort/pytest config
│   ├── .env.example                # all env vars documented, NO real values
│   └── .python-version             # 3.11
├── .github/workflows/              # test+lint workflow (NEW — generic, not GCP)
├── .gitignore
├── .gitattributes                  # eol normalization for .sh
├── .editorconfig
├── LICENSE
└── README.md                       # comprehensive
```

## App initialization (application.py)

`create_application()` returns the Flask app. ORDER MATTERS:

```
Flask() → configure_logging → configure_settings → configure_application
  → configure_database → configure_inject → configure_mail
  → register_error_handlers → register_blueprints (each w/ url_prefix='/api/v1')
  → RequestInterceptor.intercept(app)
  → register_scheduler_jobs(app)
  → return app
```

NEVER `db.create_all()` — Alembic owns the schema.

## Dependency direction
- controllers → services → models
- services → integrations
- everything → utils, configuration

NO model imports controller; NO service imports controller; NO model imports service.
