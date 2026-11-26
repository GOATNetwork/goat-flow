# Developer Quick Start

This guide connects an existing application to **hosted crypto checkout on
Testnet3**, using a TypeScript backend and browser frontend. You will create a
session for an application order, send the buyer to checkout, and read the
payment result on your backend.

The snippets are integration functions, not a complete web application. Your
application supplies login, order storage, HTTP routes, and fulfillment.
For a fixed-price Product without session-creation code, use
[Product checkout](../goatx402-checkout/README.md#product-checkout).

## 1. Prepare the merchant

In the [Testnet3 Merchant Portal](https://flow-merchant.testnet3.goat.network):

1. Register and obtain approval for an enabled DIRECT merchant.
2. Configure a receiving address for an available crypto chain/token. In updated
   portals, activate PayKit under **Payment Setup → PayKit Settings** (the
   underlying capability is still named QuickPay in the API). Check the fees
   and merchant fee-balance requirements.
3. Create API credentials under **Developer → API & Checkout Settings** and save
   the secret on your backend. Older portal labels are shown in the Merchant Guide.

The buyer needs the selected test token and native gas on the same chain.
Portal instructions are in the [Merchant Guide](merchant-guide.md).

## 2. Install and configure

Use Node.js 18 or later on the backend. Install in the corresponding applications:

```bash
# Backend
npm install goatflow-sdk-server@0.4.0
# Frontend
npm install goatflow-checkout@0.2.0
```

Set backend environment variables:

```bash
GOATX402_API_URL=https://flow-api.testnet3.goat.network
GOATX402_API_KEY=your_testnet_api_key
GOATX402_API_SECRET=your_testnet_api_secret
GOATX402_MERCHANT_ID=your_testnet_merchant_id
```

Keep the API secret out of frontend environment variables and browser bundles.
Use credentials, products, and checkout origins from the same environment;
see the [environment table](goat-flow-api-reference.md#1-origins-and-configuration).

## 3. Create and save a session on the backend

```ts
import { GoatFlowClient } from 'goatflow-sdk-server'

const flow = new GoatFlowClient({
  baseUrl: process.env.GOATX402_API_URL!,
  apiKey: process.env.GOATX402_API_KEY!,
  apiSecret: process.env.GOATX402_API_SECRET!,
})

export async function createPaymentSession(
  paymentIntentId: string,
  serverPrice: string,
) {
  return flow.createCheckoutSession({
    checkoutType: 'DIRECT',
    paymentRails: ['crypto'],
    price: serverPrice, // Decimal price from your server's cart, e.g. '9.99'.
    clientReferenceId: paymentIntentId,
  })
}
```

Connect this function to your own authenticated route, for example
`POST /api/orders/:id/checkout`. That route must:

1. Load the application order and check that the signed-in buyer owns it.
2. Read its price from your database and allocate one stable payment-intent ID.
   Serialize creation for that intent so double-clicks cannot create two payments.
3. For a new intent with no previous creation attempt, call the function once.
   Persist `checkoutId`, `url`, `expiresAt`, environment, and expected price
   against the order **before** returning `{ checkoutId }` to the browser.
   On repeat requests, check the saved attempt: resume an eligible session or
   return the known payment state. Reconcile expired, failed, or unresolved
   attempts before authorizing any replacement; do not automatically create one.

A duplicate `clientReferenceId` returns a conflict, not the previous session.
If creation times out or the response is lost, mark the attempt unresolved and
reconcile it; do not generate a new ID and immediately request payment again.
The SDK has no lookup by client reference or session-renewal helper.

This example omits return URLs. To add `successUrl` or `cancelUrl`, first
configure [Checkout return URLs](merchant-guide.md#checkout-return-urls).

## 4. Open checkout in the browser

```ts
import { GoatCheckout } from 'goatflow-checkout'

const checkout = GoatCheckout({
  origin: 'https://flow-quickpay.testnet3.goat.network',
})

export async function startPayment(businessOrderId: string) {
  const response = await fetch(
    `/api/orders/${encodeURIComponent(businessOrderId)}/checkout`,
    { method: 'POST', credentials: 'same-origin' },
  )
  if (!response.ok) throw new Error('Could not start checkout')
  const { checkoutId } = await response.json()
  if (typeof checkoutId !== 'string' || !checkoutId) {
    throw new Error('Missing checkout ID')
  }
  checkout.redirectToCheckout({ checkoutId })
}
```

Call `startPayment` from your Pay button. The route above belongs to your app;
it is the route implemented in step 3, not a GOAT Flow endpoint. Apply your
framework's normal authentication and CSRF protection to it.

The hosted page handles wallet connection and payment. A full-page redirect
works after the asynchronous session request. If using a popup instead,
`open()` must run synchronously inside a user gesture; see the
[browser SDK](../goatx402-checkout/README.md).

## 5. Read the payment result on the backend

Use the checkout ID saved in step 3, not an ID supplied by the browser. The
session's status endpoint supplies the linked `order_id` after crypto binding.
Then read that order through the authenticated server SDK:

```ts
// Same backend module as step 3. This example is for the crypto rail only.
export async function readCryptoPayment(checkoutId: string) {
  const response = await fetch(
    `https://flow-quickpay.testnet3.goat.network/checkout/v1/sessions/${encodeURIComponent(checkoutId)}/status`,
    { signal: AbortSignal.timeout(15_000), redirect: 'error', cache: 'no-store' },
  )
  if (!response.ok) throw new Error(`Checkout status failed: ${response.status}`)
  const session: { status: string; order_id?: string } = await response.json()
  if (!session.order_id) return { checkoutStatus: session.status }

  const order = await flow.getOrderStatus(session.order_id)
  if (order.merchantId !== process.env.GOATX402_MERCHANT_ID ||
      order.orderId !== session.order_id) {
    throw new Error('Payment does not match this merchant session')
  }
  const paid = order.status === 'PAYMENT_CONFIRMED' || order.status === 'INVOICED'
  const proof = paid ? await flow.getOrderProof(order.orderId) : undefined
  return { checkoutStatus: session.status, order, proof }
}
```

The raw status call is an HTTP endpoint, not an SDK method. Its response and
deployment boundary are documented in
[Checkout status](goat-flow-api-reference.md#read-checkout-status).
Treat the opaque checkout ID as a bearer capability and keep it out of logs.

Call this from a backend job or your application's authenticated order-status
route. An absent order means there is no crypto order to confirm yet; a failed
request means the result is unknown. Neither means the buyer should pay again.

Before fulfillment, compare the returned order and proof with the saved
business order: merchant/session mapping, accepted chain and token, exact
amount for that token's decimals, receiving address, and transaction identity.
The proof's `signature` is a digest, not a signed attestation; see
[Read proof](goat-flow-api-reference.md#6-read-proof).

Only then mark the business order paid and fulfill it atomically, once.
`FAILED`, `EXPIRED`, and `CANCELLED` do not authorize fulfillment. A browser
callback or return URL updates the UI only. If using webhooks, implement the
deployment-confirmed [verification contract](merchant-guide.md#9-webhook-configuration).

## 6. Verify the complete path

- Complete one test payment and find the same order in the Merchant Portal.
- Confirm your backend matches the payment terms and fulfills once.
- Refresh/retry the status read and replay a notification: fulfillment stays once.
- Test cancellation, expiry, and a lost API response without requesting a second payment.
- Before production, complete the [launch checks](merchant-guide.md#launch-checks).

For Card, use the [Card session example](goat-flow-api-reference.md#9-hosted-checkout-sessions)
and the deployment's trusted Card confirmation contract; the crypto proof flow
above does not apply. For Go, use the [Go SDK guide](../goatx402-sdk-server-go/README.md).
Other integrations are listed in the [documentation index](README.md).
