# Template — Logging filter for `logger.extra` redaction

Path: `app/utils/LoggingFilter.py`. Optional. Wire from
`Configuration.configure_logging` if `_redact`-style sanitization of
`logger.extra` is required at the application level (instead of —
or in addition to — log-pipeline-level redaction).

Closes the gap described in `patterns.md §15.2`: `RequestInterceptor`'s
`_redact` helper only scrubs the request body + query args it logs;
arbitrary `logger.info("...", extra={...})` calls bypass it.

```python
"""ExtraRedactionFilter — apply _redact to every LogRecord's `extra` dict.

WHY a logging.Filter: filters run BEFORE the formatter, on every record
regardless of which logger created it. Adding the filter to the root
logger means every `logger.info("...", extra={...})` across the app is
sanitized in one place.

WHY this is OPTIONAL: many deploys handle redaction at the log pipeline
(Vector/Fluentbit/Datadog log processors) where rules are managed by
infra/security separately from application code. If you use one of
those, you do NOT need this filter — but you also do NOT lose anything
by installing it as defense-in-depth.
"""
import logging
from typing import Any

# Import the canonical redaction helper from RequestInterceptor — single
# source of truth for the field allowlist (R67).
from app.utils.RequestInterceptor import _REDACT_FIELDS, _redact


class ExtraRedactionFilter(logging.Filter):
    """Redact secrets from every LogRecord's `extra` payload.

    The Python `logging` module merges `extra` kwargs INTO the LogRecord
    as plain attributes (not under a nested key). So we walk the record's
    `__dict__` and redact any attribute whose name matches the allowlist.
    """

    # WHY a frozenset of attribute names NOT to touch: these are the
    # built-ins Python logging always sets on every record (msg, args,
    # name, levelname, etc.). Redacting them would break formatters.
    _STD_RECORD_ATTRS: frozenset[str] = frozenset({
        "name", "msg", "args", "levelname", "levelno", "pathname",
        "filename", "module", "exc_info", "exc_text", "stack_info",
        "lineno", "funcName", "created", "msecs", "relativeCreated",
        "thread", "threadName", "processName", "process", "message",
        "asctime", "taskName",
    })

    def filter(self, record: logging.LogRecord) -> bool:
        """Mutate the record in place; return True so the record is emitted."""
        for attr_name, attr_value in list(record.__dict__.items()):
            if attr_name in self._STD_RECORD_ATTRS:
                continue
            if attr_name.lower() in _REDACT_FIELDS:
                # Top-level attribute whose name itself is sensitive.
                setattr(record, attr_name, "***")
            elif isinstance(attr_value, (dict, list)):
                # Recurse via the canonical helper — same case-insensitive
                # rules as the request-body redaction.
                setattr(record, attr_name, _redact(attr_value))
        return True


def install(root_level_logger: logging.Logger | None = None) -> None:
    """Attach the filter to the root logger (or one passed by the caller).

    Call from `Configuration.configure_logging(app, env, level)` AFTER
    `logging.config.dictConfig(config)` runs — the filter must attach
    to handlers that already exist.

    Example wiring in `utils/Logging.py`:
        from app.utils.LoggingFilter import install as install_redaction
        configure_logging(app, env, level)
        install_redaction()
    """
    target = root_level_logger or logging.getLogger()
    flt = ExtraRedactionFilter()
    for handler in target.handlers:
        handler.addFilter(flt)
```

## Wiring it into `configure_logging` (patterns.md §13)

Append to the `configure_logging` body, AFTER `dictConfig(config)`:

```python
# OPTIONAL: defense-in-depth logger.extra sanitization (see
# patterns.md §15.2). Comment out if your log pipeline handles redaction.
from app.utils.LoggingFilter import install as install_redaction
install_redaction()
```

## Checklist (reviewer enforces if the filter is installed)
- Imports `_REDACT_FIELDS` + `_redact` from `RequestInterceptor` — single
  source of truth for the allowlist (R67).
- Filter's `filter()` method ALWAYS returns True (never silently drops
  records).
- Standard LogRecord attributes are NOT redacted (would break formatters).
- Top-level attribute names are matched case-insensitively (`attr_name.lower()`).
- Nested dicts/lists go through `_redact` for recursive redaction.

## When NOT to install this filter

Skip when your log pipeline already redacts before storage. Common cases:
- Datadog "Log Processors" with redaction patterns
- Vector remap pipelines with `redact()` calls
- Fluentbit `record_modifier` filters

Documented as optional in `patterns.md §15.2` — the canonical mitigation
is log-pipeline-level redaction; this template is the app-level fallback.
