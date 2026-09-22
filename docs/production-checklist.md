# Production Checklist

Use this checklist before enabling an AfriPay integration in production.

## Account

- [ ] AfriPay Business account created
- [ ] Required account verification completed
- [ ] Production `APP_ID` received
- [ ] Production `APP_SECRET` received

## Application

- [ ] Credentials stored securely outside source control
- [ ] Payment initiation endpoint implemented
- [ ] Unique `client_token` generated for every transaction
- [ ] Pending transaction stored before checkout
- [ ] Return URL implemented
- [ ] Public webhook endpoint available
- [ ] Webhook processing is idempotent

## AfriPay configuration

- [ ] Callback URL submitted to AfriPay
- [ ] Callback URL registration confirmed

## End-to-end testing

- [ ] Checkout opens successfully
- [ ] Payment can be completed
- [ ] User returns to the application
- [ ] Webhook is received
- [ ] `client_token` matches the correct local transaction
- [ ] Provider transaction reference is stored
- [ ] Successful payment updates the local transaction
- [ ] Duplicate callback does not duplicate processing

## Production

- [ ] End-to-end production payment tested
- [ ] Logs reviewed for errors
- [ ] Integration ready for live transactions
