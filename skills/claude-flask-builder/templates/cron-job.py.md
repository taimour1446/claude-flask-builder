# Template — Cron job (no self-callback)

Path: `app/scheduler_jobs/<name>_job.py`. Register in application.py.

```python
"""Scheduled job for <purpose>. Directly calls service — no self HTTP callback (P6-8)."""
import logging
from app.configuration.Scheduler import scheduler
from app.domain.service import <Name>Service
from app.utils.distributed_lock import distributed_lock

logger = logging.getLogger(__name__)


@scheduler.task(
    'cron',
    id='<descriptive_id>',
    hour=0, minute=0,
    timezone='UTC',
    max_instances=1,   # P14-7 — required for multi-pod safety
)
def <name>_job():
    """Daily UTC midnight — <what it does>."""
    with scheduler.app.app_context():
        # Distributed lock prevents N pods running same job at once
        with distributed_lock('<descriptive_id>'):
            try:
                <Name>Service.<method>()
                logger.info('<name>_job_completed')
            except Exception:
                logger.exception('<name>_job_failed')
                raise  # let APScheduler log + reschedule
```
