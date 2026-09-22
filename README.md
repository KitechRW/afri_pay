# Afripay Payment Integration Guide

> A public, platform-agnostic reference for integrating AfriPay payments into web applications.

## Documentation

- [Getting Started](docs/getting-started.md) — account setup and the full integration lifecycle
- [Account and Credentials](docs/account-and-credentials.md) — production credentials and callback registration
- [Webhooks](docs/webhooks.md) — payment confirmation and webhook responsibilities
- [Production Checklist](docs/production-checklist.md) — go-live verification
- [Troubleshooting](docs/troubleshooting.md) — common integration problems
- [Environment Example](.env.example) — required environment variables

## Overview

This document provides a comprehensive, platform-agnostic guide for integrating Afripay into any web application. It serves as a definitive reference for how to structure your payment initialization, client redirection, return URL handling, and webhook confirmation.

Afripay uses a **form-submission checkout model** combined with asynchronous **server-to-server webhooks** to finalize transactions.

### Prerequisites

To implement this integration, you need:
1.  **Afripay Account(Business):**
   - `https://www.afripay.africa/`
2. **Afripay Credentials:**
   - `APP_ID`: Your unique Afripay application ID.
   - `APP_SECRET`: Your unique Afripay application secret.
3. **Public Webhook URL:** A publicly accessible domain to receive POST callbacks from Afripay. **Note:** You must explicitly share this callback URL with the Afripay support team so they can configure it on their end.

*(Note: If you are using WordPress, Afripay also offers a WooCommerce plugin for 100K RWF, which bypasses the need for custom integration.)*

Typically, credentials are stored in your environment variables:
```env
AFRIPAY_APP_ID=your_afripay_app_id
AFRIPAY_APP_SECRET=your_afripay_app_secret
```

---

## High-Level Architecture Flow

1. **Initiate Payment (Backend):** Your client requests to pay for a product/service. Your backend creates a `pending` payment record in your database and generates a unique order reference (`client_token`).
2. **Checkout Redirect (Frontend):** The backend responds with form data. The client creates a hidden HTML form and POSTs it to the Afripay checkout URL.
3. **User Pays (Afripay):** The user enters their payment details on the secure Afripay page.
4. **Return URL (Frontend/Backend):** After payment, Afripay redirects the user back to your site via the provided `return_url`. Your frontend polls your backend to check if the payment is complete.
5. **Webhook Confirmation (Backend):** Afripay asynchronously sends a POST request to your webhook endpoint. Your server validates the payload, updates the database, and provisions the user's purchase.

---

## Step 1: Initiating a Payment

When a user decides to checkout, your client must call your backend. Your backend prepares the payload for Afripay.

### 1.1 Generate a Unique Reference ID
You must generate a unique reference ID for every transaction. This ID serves as the `client_token` sent to Afripay, linking their transaction to your database record. 

*Note: The exact implementation below using `crypto` is **not mandatory**. You can use any method to generate a unique string, such as a standard UUID (`uuidv4()`), a sequential database ID, or a time-based string. The only requirement is that it must be unique for every order.*

```typescript
import crypto from "crypto";

// Example utility to generate a unique transaction reference
export function generateRefId(userId: string): string {
  const ts = Date.now();
  const hash = crypto.createHash("sha256").update(`${userId}-${ts}`).digest("hex").slice(0, 8);
  return `ORD-${userId.slice(-6).toUpperCase()}-${hash.toUpperCase()}`;
}
```

### 1.2 Prepare the Checkout Payload
Your backend endpoint needs to save a pending order in your database and return the required Afripay form fields to your frontend.

**Afripay Checkout URL:** `https://www.afripay.africa/checkout/index.php`

**Backend Implementation Example:**
```typescript
// POST /api/payment/initiate

const amount = 5000;  //500rwf is the minimum money you can send
const currency = "RWF"; // Supported currencies: "RWF" or "USD"
const refId = generateRefId(user.id);
const returnUrl = `https://yourdomain.com/payment/status?refid=${refId}`;

// 1. Save pending payment to your database
await db.payments.create({
  userId: user.id,
  amount: amount,
  refId: refId,
  status: "pending"
});

// 2. Build the form data for Afripay
const formData = {
  amount: String(amount),
  currency: currency,
  comment: "Payment for Order " + refId,
  client_token: refId,
  return_url: returnUrl,
  app_id: process.env.AFRIPAY_APP_ID,
  app_secret: process.env.AFRIPAY_APP_SECRET,
};

// 3. Return data to client
return response.json({
  ok: true,
  checkoutUrl: "https://www.afripay.africa/checkout/index.php",
  formData: formData
});
```

---

## Step 2: Redirecting the User

Afripay requires the client to submit a standard HTTP POST form to their checkout URL. You cannot simply redirect or make a headless API call. 

### Option A: Static HTML Form (Traditional approach)
If you are rendering a traditional server-side page, you can output a static form with the official Afripay payment button:

```html
<form action="https://www.afripay.africa/checkout/index.php" method="post" id="afripayform"> 
  <input type="hidden" name="amount" value="5000" > 
  <input type="hidden" name="currency" value="RWF" > 
  <input type="hidden" name="comment" value="Order 122"> 
  <input type="hidden" name="client_token" value="ORD-12345" > 
  <input type="hidden" name="return_url" value="https://yourdomain.com/payment/status?refid=ORD-12345" > 
  <input type="hidden" name="app_id" value="your_afripay_app_id">
  <input type="hidden" name="app_secret" value="your_afripay_app_secret"> 
  <p> 
    <!-- Official Afripay Button -->
    <input type="image" src="https://www.afripay.africa/logos/pay_with_afripay.png" alt="Pay with AfriPay" onclick="document.getElementById('afripayform').submit();">
  </p>
</form>
```

### Option B: Programmatic Submission (SPA/React/Next.js approach)
If you are building a modern frontend (e.g., React, Vue), you'll request the `formData` from your backend via an API, then dynamically generate the form and automatically submit it:

**Frontend Implementation Example:**
```javascript
const handleCheckout = async () => {
  // 1. Request initiation from your backend
  const res = await fetch("/api/payment/initiate", { method: "POST" });
  const data = await res.json();

  if (data.ok) {
    // 2. Dynamically create a form
    const form = document.createElement("form");
    form.method = "POST";
    form.action = data.checkoutUrl;

    // 3. Append all form data as hidden inputs
    Object.entries(data.formData).forEach(([key, value]) => {
      const input = document.createElement("input");
      input.type = "hidden";
      input.name = key;
      input.value = value;
      form.appendChild(input);
    });

    // 4. Append to body and submit
    document.body.appendChild(form);
    form.submit();
  }
};
```

---

## Step 3: Handling the Return URL

Once the user completes or cancels the payment on Afripay, they are redirected back to the `return_url` you specified during initiation (e.g., `https://yourdomain.com/payment/status?refid=ORD-12345`).

**Crucial Architecture Note:** The `return_url` is purely for User Experience. Do **NOT** fulfill the order or upgrade the user's account solely because they arrived at this page. Webhooks are the only secure way to confirm payment.

Since the webhook delivery (Step 4) might take a few seconds, the return page should poll your backend to check the real status of the order.

**Frontend Polling Logic:**
```javascript
// On page load for /payment/status?refid=ORD-12345
const refId = new URLSearchParams(window.location.search).get("refid");

const checkStatus = async () => {
  const res = await fetch(`/api/payment/status?refid=${refId}`);
  const data = await res.json();
  
  if (data.status === "success") {
    showSuccessScreen();
  } else if (data.status === "failed") {
    showFailureScreen();
  } else {
    // Status is still "pending". Poll again in 3 seconds.
    setTimeout(checkStatus, 3000);
  }
};

checkStatus();
```

---

## Step 4: Webhook Confirmation

The webhook is a server-to-server POST request initiated by Afripay. It securely tells your system that the payment succeeded or failed. 

**Requirements:**
1. Your webhook endpoint must be public.
2. It must be able to parse both `application/json` and `application/x-www-form-urlencoded` payloads, as payment providers can sometimes shift formats.

### Webhook Payload Specification
```json
{
  "status": "success", // Possible values: "success", "successful", "failed", "failure"
  "amount": "5000", // The amount sent
  "currency": "RWF", // RWF or USD
  "transaction_ref": "TXN_AFRIPAY_123456789", // Afripay's internal ID
  "payment_method": "momo", // mtn, airtel, visa, mastercard
  "client_token": "ORD-12345" // Maps to the refId you generated
}
```

### Webhook Implementation Example
```typescript
// POST /api/payment/webhook

export async function handleWebhook(req) {
  try {
    // 1. Extract payload safely (handle JSON or Form-Data)
    const body = extractPayload(req);
    const { status, transaction_ref, payment_method, client_token } = body;

    // 2. Look up the payment by the client_token (your refId)
    const payment = await db.payments.findOne({ refId: client_token });
    
    if (!payment) return response.status(404).send("Payment not found");

    // 3. Idempotency Check: Don't process an already completed payment
    if (payment.status !== "pending") {
      return response.status(200).send("Already processed");
    }

    // 4. Normalize Status
    const normalizedStatus = status.toLowerCase();
    const finalStatus = (normalizedStatus === "success" || normalizedStatus === "successful") 
      ? "success" 
      : "failed";

    // 5. Update Database
    await db.payments.update(payment.id, {
      status: finalStatus,
      providerRef: transaction_ref,
      method: payment_method
    });

    // 6. Fulfill the Order (e.g., give the user access)
    if (finalStatus === "success") {
      await db.users.upgrade(payment.userId);
    }

    return response.status(200).send("OK");
  } catch (error) {
    return response.status(500).send("Server Error");
  }
}
```

---

## Best Practices & Common Pitfalls

1. **Never Trust the Return URL for Fulfillment:**
   - **Pitfall:** Marking an order as "paid" just because a user lands on `/payment/status`. Users can spoof URLs.
   - **Solution:** Always rely on the server-to-server webhook (`/api/payment/webhook`) to confirm payment and grant access.

2. **Handle Duplicate Webhook Events (Idempotency):**
   - **Pitfall:** Afripay might fire the webhook multiple times for the same transaction due to network retries. This can lead to double-crediting users.
   - **Solution:** Explicitly check if your database record's `status === "pending"`. If the payment is already marked as `"success"` or `"failed"`, return a `200 OK` early without running your fulfillment logic again.

3. **Status Polling Delay:**
   - **Pitfall:** The user returns to the app from Afripay, but the webhook hasn't fired yet. If you only check the status once on page load, it might still say "pending," confusing the user.
   - **Solution:** Implement the polling mechanism demonstrated in Step 3. Show a "Confirming payment..." UI while polling until a terminal state (`success` or `failed`) is reached.

4. **Flexible Content-Type Parsing:**
   - **Pitfall:** Your webhook route crashes because it expects `application/json`, but Afripay sends `application/x-www-form-urlencoded`.
   - **Solution:** Ensure your backend middleware/framework parses both formats appropriately before attempting to destructure `status` or `client_token`.

---

## Support & Contact

If you encounter any difficulties integrating the API or need your Webhook URL configured, you can reach out directly to the Afripay technical team at **admin@afripay.africa**.
