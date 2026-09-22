# Webhooks

AfriPay payment completion should be confirmed by the server-to-server webhook, not by the browser return URL.

## Requirements

Your webhook should:

- Be publicly accessible over HTTPS.
- Accept the payload format sent by AfriPay.
- Locate the local transaction using `client_token`.
- Validate the transaction before fulfillment.
- Be idempotent so duplicate callbacks do not duplicate fulfillment.
- Store the provider transaction reference and payment method where appropriate.
- Return a successful HTTP response after the callback has been processed.

## Return URL vs webhook

The return URL is part of the user experience. A user reaching the return URL does not by itself prove that payment succeeded.

The webhook is the authoritative event used by the application to update the local payment state.

Because the user may return before the webhook arrives, the return page can poll the application's payment-status endpoint until the transaction reaches a terminal state.

See the main [Integration Guide](../README.md) for implementation examples.
