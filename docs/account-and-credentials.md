# Account and Credentials

## 1. Create an account

Create an AfriPay Business account at https://www.afripay.africa/ and complete any verification requested by AfriPay.

## 2. Prepare your callback URL

Before requesting production credentials, have a public HTTPS endpoint ready to receive payment notifications.

Example:

```text
https://example.com/api/payment/webhook
```

## 3. Request production credentials

Contact AfriPay at admin@afripay.africa and request:

- Production `APP_ID`
- Production `APP_SECRET`
- Registration of your callback/webhook URL

Example request:

```text
Subject: Request for Production AfriPay Credentials

Dear AfriPay Team,

We have completed the AfriPay integration for our application and would like to request production credentials:

- Production App ID
- Production App Secret

Please register the following callback URL:

{CALLBACK_URL}

Application/Organization: {NAME}
Website: {WEBSITE}
Purpose: {PAYMENT_PURPOSE}

Please let us know if any additional information or documentation is required.

Kind regards,
{NAME}
```

## 4. Store credentials securely

Keep credentials server-side and outside source control.

```env
AFRIPAY_APP_ID=your_afripay_app_id
AFRIPAY_APP_SECRET=your_afripay_app_secret
```

Never commit production secrets to Git.
