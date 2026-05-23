# Template — pytest endpoint test

Path: `app/tests/integration/test_<name>.py`.

```python
"""Endpoint smoke tests for /<resource>."""
import json
import pytest


def test_create_<name>_happy_path(client, authenticated_headers):
    """POST /<resource> succeeds with valid body."""
    payload = {'name': 'demo', 'amount': '10.00'}
    resp = client.post('/api/v1/<resource>',
                       data=json.dumps(payload),
                       content_type='application/json',
                       headers=authenticated_headers)
    body = resp.get_json()
    assert resp.status_code == 200
    assert body['error'] == 0
    assert body['data']['name'] == 'demo'


def test_create_<name>_missing_required(client, authenticated_headers):
    """Missing required field returns error envelope."""
    resp = client.post('/api/v1/<resource>',
                       data=json.dumps({}),
                       content_type='application/json',
                       headers=authenticated_headers)
    body = resp.get_json()
    assert body['error'] == 1
    assert 'is required' in body['message']


def test_create_<name>_requires_auth(client):
    """No token → authError."""
    resp = client.post('/api/v1/<resource>',
                       data='{}',
                       content_type='application/json')
    body = resp.get_json()
    assert body['authError'] == 1
```

Per R100 every new endpoint gets at least one happy-path + one validation-failure + one auth test.
