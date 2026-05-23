# Template — Seeder

Path: `api/seeds/<Name>Seeder.py`.

```python
"""<Name>Seeder — idempotent. Re-running must not duplicate or fail."""
import os
from flask_seeder import Seeder
from app.model.<Name> import <Name>
from app.configuration.Database import db


class <Name>Seeder(Seeder):
    def run(self):
        # Env-driven values (R-seed: NEVER hardcode credentials)
        items = [
            {'name': os.environ.get('SEED_<NAME>_FIRST', 'default-1')},
        ]
        for item in items:
            if <Name>.exists_by_name(item['name']):  # idempotency check
                continue
            <Name>.create(item)
        db.session.commit()
```
