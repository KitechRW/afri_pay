# Troubleshooting

## Payment remains pending

Check whether the webhook reached your server. The return URL should not mark a payment as successful.

Verify that:

- The callback URL is public and reachable over HTTPS.
- AfriPay has registered the correct callback URL.
- The webhook route accepts the content type being sent.
- `client_token` maps to an existing pending transaction.

## User returns before confirmation

Webhook delivery may happen after the browser returns. Keep the transaction pending and poll your application's payment-status endpoint until the webhook updates it.

## Duplicate callbacks

Webhook retries can occur. Make processing idempotent: if a transaction has already reached a terminal state, return success without running fulfillment again.

## Transaction cannot be matched

Confirm that every payment uses a unique `client_token` and that the same value is stored locally before redirecting to checkout.

## Invalid credentials

Verify that `AFRIPAY_APP_ID` and `AFRIPAY_APP_SECRET` are configured in the server environment and that production credentials are being used for production.

Do not print secrets in application logs.

## Need AfriPay assistance

For credential issuance or callback configuration, contact admin@afripay.africa.
