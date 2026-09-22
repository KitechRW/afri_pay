# AfriPay Payment Integration Guide

A public, platform-agnostic guide for integrating AfriPay payments into a web application.

AfriPay uses a **form-submission checkout flow** and **server-to-server webhooks** for payment confirmation.

## Integration Lifecycle

```text
Create AfriPay Business Account
        ↓
Complete Account Verification
        ↓
Prepare Public Webhook URL
        ↓
Request APP_ID + APP_SECRET
        ↓
Register Callback URL with AfriPay
        ↓
Configure Application
        ↓
Initiate Payment
        ↓
POST to AfriPay Checkout
        ↓
User Completes Payment
        ↓
Browser Return + Server Webhook
        ↓
Confirm Transaction
        ↓
Display Final Status
```

The lifecycle has two phases: **setup**, which is normally completed once for an application, and the **transaction flow**, which runs for every payment.

---

## 1. Setup

### 1.1 Create an AfriPay Business Account

Create a Business account at:

https://www.afripay.africa/

Complete any account or business verification required by AfriPay.

### 1.2 Prepare a Public Webhook URL

Your application needs a publicly accessible HTTPS endpoint where AfriPay can send payment notifications.

Example:

```text
https://example.com/api/payment/webhook
```

The callback URL must be shared with AfriPay so it can be configured for your account.

### 1.3 Request Production Credentials

Contact **admin@afripay.africa** and request:

- Production `APP_ID`
- Production `APP_SECRET`
- Registration of your callback/webhook URL

A reusable request template is available in [Account and Credentials](docs/account-and-credentials.md).

### 1.4 Configure Environment Variables

Keep credentials server-side and outside source control.

```env
AFRIPAY_APP_ID=your_afripay_app_id
AFRIPAY_APP_SECRET=your_afripay_app_secret
```

See [.env.example](.env.example).

---

## 2. How a Payment Works

For every transaction:

1. The backend generates a unique `client_token` and stores a local payment as `pending`.
2. The application POSTs the checkout fields to AfriPay.
3. The user completes payment on the AfriPay checkout page.
4. AfriPay redirects the browser to the supplied `return_url`.
5. AfriPay sends the payment result to the registered webhook.
6. The backend matches the callback to the local transaction using `client_token`.
7. The backend updates the payment and performs any fulfillment.
8. The return page displays the final status after the backend confirms it.

> **Important:** The `return_url` is for user experience. Do not treat a browser return as proof of payment. Use the server-to-server webhook to confirm the transaction.

```text
                         ┌── Browser → return_url ──┐
                         │                          ↓
User → Application → AfriPay Checkout         Status Page
                         │                          ↑
                         └── Webhook → Backend ─────┘
                                      ↓
                              Update Transaction
```

---

## 3. Initiate a Payment

### 3.1 Generate a Unique `client_token`

Every transaction needs a unique reference that links the AfriPay transaction to the payment stored by your application.

The format is application-defined. A UUID, database-generated reference, or another guaranteed-unique value can be used.

Example:

```typescript
import crypto from "crypto";

export function generateRefId(userId: string): string {
  const ts = Date.now();
  const hash = crypto
    .createHash("sha256")
    .update(`${userId}-${ts}`)
    .digest("hex")
    .slice(0, 8);

  return `ORD-${userId.slice(-6).toUpperCase()}-${hash.toUpperCase()}`;
}
```

### 3.2 Store the Pending Payment

Create the local payment **before** sending the user to AfriPay.

```typescript
const amount = 5000;
const currency = "RWF";
const refId = generateRefId(user.id);

await db.payments.create({
  userId: user.id,
  amount,
  currency,
  refId,
  status: "pending"
});
```

### 3.3 Build the Checkout Payload

**Checkout URL**

```text
https://www.afripay.africa/checkout/index.php
```

Example backend response:

```typescript
const returnUrl =
  `https://example.com/payment/status?refid=${refId}`;

const formData = {
  amount: String(amount),
  currency,
  comment: "Payment for Order " + refId,
  client_token: refId,
  return_url: returnUrl,
  app_id: process.env.AFRIPAY_APP_ID,
  app_secret: process.env.AFRIPAY_APP_SECRET
};

return response.json({
  ok: true,
  checkoutUrl: "https://www.afripay.africa/checkout/index.php",
  formData
});
```

---

## 4. Submit to AfriPay Checkout

AfriPay checkout is opened by submitting an HTTP POST form containing the payment fields.

For a modern frontend, the form can be created programmatically:

```javascript
const handleCheckout = async () => {
  const res = await fetch("/api/payment/initiate", {
    method: "POST"
  });

  const data = await res.json();

  if (!data.ok) return;

  const form = document.createElement("form");
  form.method = "POST";
  form.action = data.checkoutUrl;

  Object.entries(data.formData).forEach(([key, value]) => {
    const input = document.createElement("input");
    input.type = "hidden";
    input.name = key;
    input.value = value;
    form.appendChild(input);
  });

  document.body.appendChild(form);
  form.submit();
};
```

A traditional server-rendered application can use a normal HTML form with the same fields.

---

## 5. Handle the Return URL

After checkout, AfriPay redirects the browser to the `return_url` supplied during payment initiation.

The page should query your backend for the transaction status rather than assuming payment succeeded.

Example:

```javascript
const refId =
  new URLSearchParams(window.location.search).get("refid");

const checkStatus = async () => {
  const res = await fetch(
    `/api/payment/status?refid=${refId}`
  );

  const data = await res.json();

  if (data.status === "success") {
    showSuccessScreen();
    return;
  }

  if (data.status === "failed") {
    showFailureScreen();
    return;
  }

  setTimeout(checkStatus, 3000);
};

checkStatus();
```

Polling is useful because the browser may return before the webhook has been processed.

---

## 6. Handle the Webhook

The webhook is the server-to-server payment notification sent by AfriPay.

Your endpoint should:

- Be publicly accessible.
- Parse the callback payload.
- Find the local transaction using `client_token`.
- Avoid processing an already completed transaction twice.
- Store the AfriPay transaction reference and payment method.
- Update the local transaction status.
- Run fulfillment only after successful confirmation.
- Return a successful HTTP response after processing.

Example payload:

```json
{
  "status": "success",
  "amount": "5000",
  "currency": "RWF",
  "transaction_ref": "TXN_AFRIPAY_123456789",
  "payment_method": "momo",
  "client_token": "ORD-12345"
}
```

Example handler:

```typescript
export async function handleWebhook(req) {
  try {
    const body = extractPayload(req);

    const {
      status,
      transaction_ref,
      payment_method,
      client_token
    } = body;

    const payment = await db.payments.findOne({
      refId: client_token
    });

    if (!payment) {
      return response.status(404).send("Payment not found");
    }

    // Idempotency: do not fulfill the same payment twice.
    if (payment.status !== "pending") {
      return response.status(200).send("Already processed");
    }

    const normalizedStatus = status.toLowerCase();

    const finalStatus =
      normalizedStatus === "success" ||
      normalizedStatus === "successful"
        ? "success"
        : "failed";

    await db.payments.update(payment.id, {
      status: finalStatus,
      providerRef: transaction_ref,
      method: payment_method
    });

    if (finalStatus === "success") {
      await fulfillPayment(payment);
    }

    return response.status(200).send("OK");
  } catch (error) {
    return response.status(500).send("Server Error");
  }
}
```

See [Webhooks](docs/webhooks.md) for focused webhook guidance.

---

## 7. Production Verification

Before considering the integration live, verify the entire flow with a real production transaction:

- Account and credentials are active.
- Callback URL is registered.
- Checkout opens successfully.
- A transaction is created as `pending`.
- Payment can be completed.
- The browser returns correctly.
- The webhook reaches the application.
- `client_token` resolves to the correct local transaction.
- The provider transaction reference is stored.
- Successful payment triggers fulfillment once.
- Duplicate callbacks do not duplicate fulfillment.
- The final status is shown correctly to the user.

Use the complete [Production Checklist](docs/production-checklist.md).

---

## 8. Common Integration Issues

**Payment stays pending:** verify that the webhook reached your server, the callback URL registered with AfriPay is correct, and `client_token` matches a local transaction.

**User returns before confirmation:** keep the transaction pending and poll the backend while waiting for the webhook.

**Duplicate callbacks:** make webhook processing idempotent so fulfillment cannot run twice.

**Transaction cannot be matched:** ensure every transaction receives a unique `client_token` and that it is stored before checkout.

**Invalid credentials:** verify the production `APP_ID` and `APP_SECRET` in the server environment.

See [Troubleshooting](docs/troubleshooting.md) for more details.

---

## Documentation

The README is designed to be enough for a standard integration. Additional focused references are available when needed:

- [Getting Started](docs/getting-started.md)
- [Account and Credentials](docs/account-and-credentials.md)
- [Webhooks](docs/webhooks.md)
- [Production Checklist](docs/production-checklist.md)
- [Troubleshooting](docs/troubleshooting.md)

## Support

For AfriPay account credentials or callback configuration, contact **admin@afripay.africa**.
