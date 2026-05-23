# Template — Stripe webhook handler

Path: `app/api/StripeWebhookController.py`.

```python
"""Stripe webhook receiver with signature verification (R70)."""
import logging
import stripe
from flask import Blueprint, request
from app.utils.BaseResponse import BaseResponse
from app.utils.Settings import settings

logger = logging.getLogger(__name__)


def stripe_webhook_blueprint() -> Blueprint:
    bp = Blueprint('stripe_webhook', __name__)

    @bp.route('/webhook/stripe', methods=['POST'])
    def webhook() -> BaseResponse:
        """Receive Stripe events. Signature MUST verify or we reject."""
        payload = request.get_data(as_text=True)
        sig_header = request.headers.get('Stripe-Signature', '')

        try:
            event = stripe.Webhook.construct_event(
                payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
            )
        except (ValueError, stripe.error.SignatureVerificationError):
            logger.warning('stripe_webhook_invalid_signature')
            return BaseResponse.respondError(message='Invalid signature')

        # Route by event type — keep handler pure, delegate to service
        # ...
        return BaseResponse.respond({'ok': True})

    return bp
```
