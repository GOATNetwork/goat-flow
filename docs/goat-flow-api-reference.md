# GOAT Flow API Reference

This reference describes the API and wire behavior used by the TypeScript and
Go server SDKs, browser SDK, QuickPay package, and GOAT Flow MPP adapter.

For a tutorial, start with the
[Developer Quick Start](./goat-flow-developer-quickstart.md).
Use this page to look up a specific contract or limitation.

| Look up | Section |
| --- | --- |
| Environment URLs | [Origins](#1-origins-and-configuration) |
| Signing and error handling | [HMAC](#2-hmac-authentication) · [Errors and retries](#13-error-model) |
| Hosted checkout | [Session fields](#9-hosted-checkout-sessions) · [Status](#read-checkout-status) · [Redirect settings](#checkout-redirect-settings) |
| Custom orders | [Create](#4-create-order) · [Status](#5-read-order-status) · [Proof](#6-read-proof) · [Cancel](#7-cancel-order) |
| Public payments | [QuickPay / PayKit](#10-quickpay) · [MPP](advanced/mpp.md) |
| Installed package support | [Versions and limitations](#14-runtime-capability-and-version-sources) |

## 1. Origins and configuration

```bash
GOATX402_API_URL=https://flow-api.goat.network
GOATX402_API_KEY=your_api_key
GOATX402_API_SECRET=your_api_secret
GOATX402_MERCHANT_ID=your_merchant_id
```

| Surface | Testnet3 origin | Mainnet origin |
| --- | --- | --- |
| GOAT Flow product website | — | `https://www.goat.network/flow` |
| Merchant Portal | `https://flow-merchant.testnet3.goat.network` | `https://flow-merchant.goat.network` |
| Admin Portal (authorized operators only) | `https://flow-admin.testnet3.goat.network` | `https://flow-admin.goat.network` |
| Flow API / standalone MPP Core | `https://flow-api.testnet3.goat.network` | `https://flow-api.goat.network` |
| Public QuickPay / Hosted Checkout and same-origin API | `https://flow-quickpay.testnet3.goat.network` | `https://flow-quickpay.goat.network` |

The merchant API URL is configurable in both server SDKs. Test and private
deployments may use different origins. Merchant integrations do not call the
Admin Portal.

The product website is informational; it is not an SDK `baseUrl` or Checkout
`origin`. GOAT Testnet3 is chain `48816` (`eip155:48816`) and GOAT Mainnet is
chain `2345` (`eip155:2345`). Other enabled EVM chains and tokens come from the
target deployment. Package versions do not select the deployment, and Stripe
test/live mode is independent of the chain ID.

The QuickPay client derives its public session and MPP paths from the trusted
QuickPay link origin. It does not switch to `flow-api` from manifest endpoint
fields. Use `flow-api` for authenticated merchant API calls and explicitly
configured standalone MPP.

Keep merchant accounts, API/webhook secrets, products, receiving configuration,
fee balances, and payment records separate between environments. Verify
production configuration from the live deployment; Testnet3 screenshots are
test evidence only.

## 2. HMAC authentication

Protected merchant endpoints require:

- `X-API-Key`
- `X-Timestamp`
- `X-Nonce`
- `X-Sign`

The server SDK algorithm is:

1. Convert top-level body values to strings.
2. Add `api_key`, `timestamp` (Unix seconds), and `nonce`.
3. Remove `sign` and empty-string values.
4. Sort keys lexicographically.
5. Join as `key=value&key=value`.
6. HMAC-SHA256 with the API secret and hex encode.

This legacy delimiter format is not injective for arbitrary values containing
`&` or `=`. Do not treat it as safely binding unrestricted scalar input. A
complete correction requires a versioned canonicalization contract deployed
together in Core and both server SDKs; changing only a client would make its
signatures incompatible with the current verifier. Until that migration is
complete, the active deployment must enforce an identical allowed-value policy
before authentication and request handling.

Current authenticated SDK `GET` methods have no query parameters and therefore
sign only the authentication fields.

The signing format is scalar-only. Hosted Checkout arrays/maps are sent as JSON
strings:

- `acceptable_tokens`
- `line_items_json`
- `public_metadata_json`
- `private_metadata_json`

Do not expose the API secret or HMAC code in the browser.
For authentication errors, see [Merchant API 401](goat-flow-faq.md#the-merchant-api-returned-http-401).

## 3. Merchant endpoint summary

The current public merchant contract uses DIRECT. Operator callback rows are
compatibility reference for environments explicitly provisioned by the GOAT
operator; their presence in an SDK does not enable them for a merchant.

| Method | Endpoint | Auth | Success |
| --- | --- | --- | --- |
| Create order | `POST /api/v1/orders` | HMAC | `402` |
| Create Hosted Checkout Session | `POST /api/v1/checkout/sessions` | HMAC | `200` |
| Read order | `GET /api/v1/orders/{order_id}` | HMAC | `200` |
| Read proof | `GET /api/v1/orders/{order_id}/proof` | HMAC | `200` |
| Submit operator callback signature | `POST /api/v1/orders/{order_id}/calldata-signature` | HMAC | `200` |
| Cancel order | `POST /api/v1/orders/{order_id}/cancel` | HMAC | `200` |
| Read merchant | `GET /merchants/{merchant_id}` | Public | `200` |

HTTP `402` is a success only where the endpoint defines a payment challenge
(create order and the GOAT Flow MPP profile's challenge endpoint).

Both current server SDKs accept `402` as success only for order creation.
Status, proof, checkout, signature, and cancellation calls fail closed on an
unexpected `402`.

## 4. Create order

```http
POST /api/v1/orders
```

### Request

| JSON field | Type | Required | TypeScript | Go |
| --- | --- | --- | --- | --- |
| `dapp_order_id` | string | Yes | `dappOrderId` | `DappOrderID` |
| `chain_id` | integer | Yes | `chainId` | `ChainID` |
| `token_symbol` | string | Yes | `tokenSymbol` | `TokenSymbol` |
| `token_contract` | string | No | `tokenContract` | `TokenContract` |
| `from_address` | string | Yes | `fromAddress` | `FromAddress` |
| `amount_wei` | integer string | Yes | `amountWei` | `AmountWei` |
| `callback_calldata` | hex string | No | `callbackCalldata` | `CallbackCalldata` |

An ordinary duplicate `dapp_order_id` is rejected; order creation does not
return the earlier order. Persist the successful `order_id` and reconcile it
after an ambiguous timeout or duplicate response. The TypeScript-only
`recoverExistingOrder` field is restricted to exact, service-recognized
`topup:` retries and is not a general merchant idempotency option.

### Raw response

```http
HTTP/1.1 402 Payment Required
Content-Type: application/json
PAYMENT-REQUIRED: <base64-x402-json>
```

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://flow-api.goat.network/api/v1/orders/{order_id}",
    "description": "Payment",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:2345",
      "amount": "10000000",
      "asset": "0xToken",
      "payTo": "0xMerchant",
      "maxTimeoutSeconds": 600,
      "extra": {
        "flow": "ERC20_DIRECT",
        "tokenSymbol": "USDC"
      }
    }
  ],
  "extensions": {
    "goatx402": {
      "destinationChain": "eip155:2345",
      "expiresAt": 1780000000,
      "paymentMethod": "transfer",
      "receiveType": "DIRECT"
    }
  },
  "order_id": "order-id",
  "flow": "ERC20_DIRECT",
  "token_symbol": "USDC"
}
```

The documented flow is `ERC20_DIRECT`. The browser transfers the ERC-20 token
to the merchant receiving address returned as `payTo`.

Operator-provisioned callback fields and signature submission are retained as
compatibility reference in [Appendix A](#appendix-a-operator-provisioned-callback-compatibility).

### Normalized server `Order`

`createOrder()` normalizes the first `accepts[]` entry:

| Server field | Source |
| --- | --- |
| `orderId` | `order_id` |
| `flow` | top-level `flow`, then `accepts[0].extra.flow`, default `ERC20_DIRECT` in TS |
| `tokenSymbol` | top-level `token_symbol`, then `accepts[0].extra.tokenSymbol` |
| `tokenContract` | `accepts[0].asset` |
| `payToAddress` | `accepts[0].payTo` |
| `fromChainId` | parsed from `accepts[0].network` |
| `payToChainId` | parsed from `extensions.goatx402.destinationChain` |
| `amountWei` | `accepts[0].amount` |
| `expiresAt` | `extensions.goatx402.expiresAt` |
| `calldataSignRequest` | top-level `calldata_sign_request` |

These are normalization outputs, not response-validation guarantees. The
TypeScript client can fall back to request values for token, source chain, and
amount and defaults a missing flow to `ERC20_DIRECT`; the Go client only falls
back to the requested source chain. Either client can therefore expose empty or
zero values when a deployment returns an incomplete challenge. Validate every
field needed by the browser before presenting a transfer.

The browser `Order` instead requires `chainId` and `fromAddress`. Map explicitly:

```ts
function toClientOrder(serverOrder: ServerOrder, fromAddress: string): ClientOrder {
  return {
    orderId: serverOrder.orderId,
    flow: serverOrder.flow,
    tokenSymbol: serverOrder.tokenSymbol,
    tokenContract: serverOrder.tokenContract,
    fromAddress,
    payToAddress: serverOrder.payToAddress,
    chainId: serverOrder.fromChainId,
    amountWei: serverOrder.amountWei,
    expiresAt: serverOrder.expiresAt,
    calldataSignRequest: serverOrder.calldataSignRequest,
  }
}
```

## 5. Read order status

```http
GET /api/v1/orders/{order_id}
```

Response fields mapped by the SDKs:

| Wire field | TypeScript field | Go field |
| --- | --- | --- |
| `order_id` | `orderId` | `OrderID` |
| `merchant_id` | `merchantId` | `MerchantID` |
| `dapp_order_id` | `dappOrderId` | `DappOrderID` |
| `chain_id` | `chainId` | `ChainID` |
| `token_contract` | `tokenContract` | `TokenContract` |
| `token_symbol` | `tokenSymbol` | `TokenSymbol` |
| `from_address` | `fromAddress` | `FromAddress` |
| `amount_wei` | `amountWei` | `AmountWei` |
| `status` | `status` | `Status` |
| `tx_hash` | `txHash` | `TxHash` |
| `confirmed_at` | `confirmedAt` | `ConfirmedAt` |

Current SDK status values:

- `CHECKOUT_VERIFIED`
- `PAYMENT_CONFIRMED`
- `INVOICED`
- `FAILED`
- `EXPIRED`
- `CANCELLED`

`PAYMENT_CONFIRMED` and `INVOICED` are successful terminal states for the
Server SDK order waiters. Core can advance a DIRECT order from
`PAYMENT_CONFIRMED` to `INVOICED` inside one watcher transaction, so a poller
may observe only `INVOICED`.

The polling helpers differ in timing and retry policy. TypeScript reads
immediately, retries network failures, request timeouts, `408`, `429`, and
server errors, and surfaces other deterministic 4xx errors. Each request has a
30-second deadline bounded by the remaining overall timeout. Go waits one
interval before its first read and retries all status-read errors, including
`401`, `403`, and `404`, until a later successful poll, timeout, or context
cancellation. Its default HTTP client has a 30-second timeout. The Go waiter's
timeout is checked at interval ticks and does not bound an in-flight HTTP
request. Supply a context deadline to bound the wait, or implement explicit
`GetOrderStatus` polling to also surface deterministic errors immediately.

## 6. Read proof

```http
GET /api/v1/orders/{order_id}/proof
```

```json
{
  "payload": {
    "order_id": "order-id",
    "tx_hash": "0x...",
    "log_index": 0,
    "from_addr": "0x...",
    "to_addr": "0x...",
    "amount_wei": "10000000",
    "from_chain_id": 2345,
    "status": "INVOICED"
  },
  "signature": "0x..."
}
```

Retrieve proof after a trusted successful order status.

The historical field name `signature` is misleading: this value is not a
signature or attestation. It is Keccak256 over these seven payload fields,
concatenated without separators in this exact order:

```text
order_id || tx_hash || log_index || from_addr || to_addr || amount_wei || from_chain_id
```

The digest does not cover `status` or any other field. Anyone can recompute it.
Verify `payload.tx_hash` on-chain when independent proof is required.

## 7. Cancel order

```http
POST /api/v1/orders/{order_id}/cancel
Content-Type: application/json

{}
```

The SDK contract permits cancellation while the order is
`CHECKOUT_VERIFIED`. Do not assume another state is cancellable.

## 8. Read merchant

```http
GET /merchants/{merchant_id}
```

This endpoint is public.

The TypeScript client currently expects a wire response with `merchant_id`,
optional `name`/`logo`, `receive_type` (`DIRECT`), and `wallets[]`, then maps
wallets to `supportedTokens`.

The Go `MerchantInfo` type currently declares `supported_tokens` directly.
Because these two clients expect different token-list field names,
verify the response shape of your target deployment before relying on the Go
`SupportedTokens` field. If the response contains `wallets[]`, an empty Go
field does not prove that the merchant has no configured tokens.

## 9. Hosted Checkout Sessions

```http
POST /api/v1/checkout/sessions
```

Authenticated with merchant HMAC. The API key determines the merchant.

### Request

Create a DIRECT session:

```json
{
  "checkout_type": "DIRECT",
  "price": "9.99",
  "client_reference_id": "cart-123",
  "line_items_json": "[{\"name\":\"Mug\",\"amount\":\"9.99\"}]"
}
```

The public DIRECT field mapping is:

| Wire field | TypeScript | Go | Use |
| --- | --- | --- | --- |
| `checkout_type` | `checkoutType` | `CheckoutType` | Use `DIRECT` for public merchant checkout |
| `price` | `price` | `Price` | Decimal product or cart price |
| `success_url` | `successUrl` | `SuccessURL` | Optional allowlisted success redirect |
| `cancel_url` | `cancelUrl` | `CancelURL` | Optional allowlisted cancel redirect |
| `client_reference_id` | `clientReferenceId` | `ClientReferenceID` | Optional unique merchant correlation reference; duplicates conflict rather than returning the original session |
| `expires_in` | `expiresIn` | `ExpiresIn` | Optional lifetime in seconds |
| `line_items_json` | `lineItems` | `LineItems` | JSON-stringified display items |
| `public_metadata_json` | `publicMetadata` | `PublicMetadata` | JSON-stringified public metadata |
| `private_metadata_json` | `privateMetadata` | `PrivateMetadata` | JSON-stringified merchant-only metadata |
| `payment_rails` | `paymentRails` | Not available | `('crypto' \| 'fiat')[]`, serialized as CSV; defaults to Crypto |
| `fiat_currency` | `fiatCurrency` | Not available | ISO-4217 code such as `USD`; actual support is merchant/deployment-specific |
| `fiat_amount` | `fiatAmount` | Not available | Positive decimal string such as `9.99`; not wei or integer minor units |

Use the server SDK so nested values are serialized consistently with HMAC.

The authenticated merchant must match `checkout_type`. For DIRECT, offering
Crypto requires QuickPay to be enabled; a fiat-only authenticated session does
not. Fiat eligibility and a configured Stripe runtime are still required.
Public QuickPay Card-link creation has its own QuickPay enablement check.

Card-only TypeScript example using `goatflow-sdk-server@0.4.0` and the
Testnet3 API:

```ts
import { GoatFlowClient } from 'goatflow-sdk-server'

const client = new GoatFlowClient({
  baseUrl: 'https://flow-api.testnet3.goat.network',
  apiKey: process.env.GOATX402_API_KEY!,
  apiSecret: process.env.GOATX402_API_SECRET!,
})

const session = await client.createCheckoutSession({
  checkoutType: 'DIRECT',
  paymentRails: ['fiat'],
  fiatCurrency: 'USD',
  fiatAmount: '9.99',
  clientReferenceId: 'your-persisted-payment-intent-id',
})
```

Confirm Stripe test mode, merchant fiat eligibility, and connected-account
status before testing. A Card-only session omits `price`; a two-rail session
also supplies the Crypto `price`. The service does not infer or convert between
the two amounts. `checkoutType: 'DIRECT'` does not mean Crypto-only.

The current public Go helper does not expose the three Card request fields.
Do not translate this example directly to Go or treat an unrecognized struct
field as supported. Use the TypeScript SDK or a deployment-confirmed HTTP
contract until a later Go SDK version adds them.

Operator-provisioned fields, deprecated wrappers, and callback trust boundaries
are isolated in
[Appendix A](#appendix-a-operator-provisioned-callback-compatibility). They are
not part of public merchant onboarding.

### Response

For the Testnet3 Card session above, a response has this shape:

```json
{
  "checkout_id": "cs_...",
  "checkout_type": "DIRECT",
  "url": "https://flow-quickpay.testnet3.goat.network/paykit/direct?cs=cs_...",
  "expires_at": 1893456000
}
```

The exported `CheckoutSession.checkoutType` / Go `CheckoutType` response field
is typed as `string`. Known current values are `DIRECT` and `DELEGATE`; handle an
unknown future value explicitly.

The backend saves the session, checks the returned URL against its configured
Checkout origin, and sends `{ checkoutId, url }` to the browser. Keep API
credentials on the backend. With these validated values:

```ts
const goat = GoatCheckout({ origin: new URL(session.url).origin })
goat.open({ checkoutId: session.checkoutId })
```

The public page normally owns:

- `GET /checkout/v1/sessions/{checkout_id}`
- `GET /checkout/v1/sessions/{checkout_id}/status`
- `POST /checkout/v1/sessions/{checkout_id}/bind`
- `POST /checkout/v1/sessions/{checkout_id}/signature` for an
  operator-provisioned callback signature when required

Treat the checkout ID as a bearer capability. Browser `onSuccess` is not proof
for fulfillment.

Persist the successful `checkout_id` and `url`. A repeated non-empty
`client_reference_id` for the same merchant returns a conflict; it is not a
request to recover the original opaque handle.

In this HMAC API, the merchant backend supplies `client_reference_id`.
References accepted through public Product/QuickPay checkout entry points can
instead be buyer-supplied. A reference alone proves neither merchant
authorization nor payment; fulfill using your saved business-order/session
mapping and verified payment terms and status.

The current public SDKs do not expose submission protocol 1 or a checkout
renewal method; see the [published SDK boundary](#published-sdk-boundary).

### Read checkout status

```http
GET /checkout/v1/sessions/{checkout_id}/status
```

Call the trusted Checkout origin for the saved session's environment. This is
a public, bearer-handle endpoint, not a merchant HMAC method. The current Core
implementation returns these fields; verify endpoint availability on the target
deployment before integrating:

| Field | Meaning |
| --- | --- |
| `status` | Checkout session state |
| `order_id` | Linked order ID, omitted before an order is bound |
| `order_status` | Linked order's state, when available |
| `tx_hash` | Linked order's transaction hash, when available |

| Session `status` | Meaning |
| --- | --- |
| `OPEN` | Session is open. A Card order can already be linked and processing or held. |
| `BOUND` | A crypto order is linked to the session. |
| `SIGNED` | A signature was submitted in an operator-provisioned callback flow. Public DIRECT checkout skips this step. |
| `COMPLETED` | The linked order reached `INVOICED`. Verify the purchase and fulfillment eligibility before delivery. |
| `EXPIRED` | The payment window expired. Reconcile any transfer already submitted. |
| `CANCELLED` | The session or linked order was cancelled. |
| `FAILED` | The linked order failed. Withhold fulfillment. |

For crypto, resolve `order_id` from your persisted checkout handle, then use
the authenticated [order status](#5-read-order-status) and [proof](#6-read-proof)
APIs. Match the expected business payment terms before fulfilling once.
Use `order_status` when checking an order and `status` when checking a session.
An absent order ID, network error, or delayed status is not proof of non-payment.
The crypto order/proof procedure does not define Card confirmation; use the
deployment's trusted Card status/webhook contract.

### Checkout redirect settings

These are portal configuration endpoints, not merchant HMAC APIs or methods
of the published server SDK. They describe Core Checkout settings support
reviewed on November 26, 2026; confirm that support in the target deployment.

| Endpoint | Authentication | Read | Write |
| --- | --- | --- | --- |
| `GET/PUT /merchant/v1/checkout/settings` | Merchant access token | Owner/member | Owner |
| `GET/PUT /admin/merchants/{merchant_id}/checkout/settings` | Admin session with completed MFA | Viewer/operator/super_admin | Operator/super_admin; CSRF checks apply |

GET and successful PUT return this shape; PUT accepts the same shape:

```json
{
  "redirect_allowlist": ["https://shop.example.com/checkout"]
}
```

PUT replaces the entire list. Send `{"redirect_allowlist":[]}` to clear it;
omitted/null fields, wrong types, unknown fields, or invalid entries return
`400`. A body larger than 16 KiB returns `413`. The merchant route derives the
merchant from the access token, not the body. These settings do not enable a
payment method and do not require QuickPay to be enabled.

The settings contract accepts at most 20 entries, each at most 512 bytes both
before and after URL encoding. Protocol, host, and port must match. A path
prefix matches itself and child paths: `/checkout` allows `/checkout/success`,
not `/checkout-other`. HTTP is allowed only for `localhost`, `127.0.0.1`, or
`[::1]`. Entries cannot contain wildcards, credentials, query strings,
fragments, or ambiguous paths; internationalized hostnames use Punycode.
Matching uses the encoded browser pathname, so preserve path case and percent
encoding. The actual success/cancel URL may contain a query or fragment.

An empty allowlist rejects new sessions containing either return URL with
HTTP `400`. Changes apply to new sessions; existing authenticated sessions
retain the return URLs approved at creation. Public product/custom URL
parameters are instead checked against the current discovery allowlist.
The setting is independent of enabling QuickPay or Card payments. It is not
CORS, a webhook destination, or a Stripe OAuth callback setting.

For portal steps, see [Checkout return URLs](merchant-guide.md#checkout-return-urls).
Use merchant HMAC credentials to create sessions; settings use portal authentication.

## 10. QuickPay

QuickPay links are public and same-origin:

| Surface | Endpoint |
| --- | --- |
| Web/agent entry | `GET /paykit/{merchant_id}` (legacy `/quickpay/{merchant_id}`) |
| Agent instructions | `GET /paykit/{merchant_id}/agent.md` (legacy `/quickpay/{merchant_id}/agent.md`) |
| Manifest | `GET /paykit/{merchant_id}/manifest.json` (legacy `/quickpay/{merchant_id}/manifest.json`) |
| Discovery | `GET /quickpay/v1/merchants/{merchant_id}` |
| Create x402 session | `POST /quickpay/v1/x402/sessions` |
| Read x402 session | `GET /quickpay/v1/x402/sessions/{session_id}` |
| Create hosted card checkout link | `POST /quickpay/v1/fiat/sessions` |

The `goatflow-paykit` package accepts canonical `/paykit/{merchant_id}` and
legacy `/quickpay/{merchant_id}` links, optionally ending in `/agent.md` or
`/manifest.json`, over HTTPS (or HTTP loopback for local development). It
derives all called endpoints from that trusted origin; the API prefix remains
`/quickpay/v1`. See [release status and migration](goat-flow-api-reference.md#14-runtime-capability-and-version-sources).

A `client_reference_id` supplied through a public QuickPay endpoint is
buyer-controlled correlation data. It does not prove merchant authorization,
business-order ownership, or payment and must not authorize fulfillment alone.

### Read a merchant manifest

Example fields from `GET /paykit/acme/manifest.json`. The token address and
limits are illustrative. Read the merchant's manifest for the selected environment.

```json
{
  "schema": "goatx402.quickpay.v1",
  "merchant": { "merchant_id": "acme" },
  "rails": {
    "x402": {
      "enabled": true,
      "custom_amount": true,
      "tokens": [{
        "chain_id": 48816,
        "token_symbol": "USDC",
        "token_contract": "0x1111111111111111111111111111111111111111",
        "decimals": 6,
        "min_amount_wei": "1000000",
        "max_amount_wei": "25000000"
      }]
    },
    "mpp": { "enabled": false, "routes": [] },
    "fiat": {
      "enabled": true, "custom_amount": true, "human_action_required": true,
      "currency": "USD", "minor_unit_exponent": 2
    }
  }
}
```

The limits apply to that token in base units, so this example allows 1 to 25
tokens. `max_amount_wei` is optional. The fiat rail creates a hosted Card link
for the buyer to complete.

### Create public x402 session

```json
{
  "merchant_id": "merchant_123",
  "payer_addr": "0xUser",
  "chain_id": 2345,
  "token_contract": "0xToken",
  "amount_wei": "10000000",
  "memo": "invoice-123",
  "idempotency_key": "invoice-123:user-456"
}
```

Use either:

- `amount_wei` for custom amount
- `product_key` for a server-priced product

That JSON is the raw public API shape. The library uses camelCase options:
`amount`, `tokenSymbol`/`tokenContract`, `chainId`, `memo`, and
`idempotencyKey`; it derives `merchant_id` from the trusted QuickPay URL and
`payer_addr` from the payment backend. Product mode uses `productKey`. The
current library does not expose a `clientReferenceId` option, so do not pass raw
snake_case fields to `QuickPayClient.payX402()` or `payProduct()`.

The package validates the trusted origin, merchant identity, and the individual
token, Product, or route entry selected for a payment. Current boundaries:

- non-array `tokens` or `routes` values are normalized to empty lists rather
  than rejected as a malformed manifest;
- for a fresh call without an explicit idempotency key, `payX402()` rejects a
  disabled x402 rail or `custom_amount: false`; an explicit key allows the
  server to attempt recovery before checking its current rail settings; and
- Product min/max enforcement remains server-authoritative even when the client
  performs local price and token checks.

Treat manifest validation as client-side discovery and preflight, not a
replacement for server validation.

### Create a hosted Card link with PayKit

`QuickPayClient.createFiatCheckoutLink(options)` calls
`POST /quickpay/v1/fiat/sessions` on the trusted PayKit link origin. The client
derives `merchant_id` from that link; it does not require a merchant API secret
or a wallet signer to create the link.

| SDK option | Sent JSON field | Use |
| --- | --- | --- |
| `productKey` | `product_key` | Fixed-price Product advertised under `rails.fiat.products`; choose either this or `amount` |
| `amount` | `amount` | Positive decimal string in the manifest's Card currency, such as `10.00`; precision follows `minor_unit_exponent` |
| `memo` | `memo` | Custom-amount reference; required when the manifest sets `memo_required`; a non-empty memo is rejected in Product mode |

Exactly one of `productKey` or `amount` must be provided. Currency comes from
the manifest. The standalone `createFiatCheckoutLink()` function also takes
`input`, the trusted PayKit URL; the client method uses its constructor URL.

```ts
import { QuickPayClient } from 'goatflow-paykit'

const client = new QuickPayClient(
  'https://flow-quickpay.testnet3.goat.network/paykit/merchant_123/agent.md',
)

const cardLink = await client.createFiatCheckoutLink({ productKey: 'mug' })
// Return cardLink.url to the payer so they can complete hosted payment.
```

The SDK returns a `FiatCheckoutLink` object with these field names:

| SDK result field | Type | Meaning |
| --- | --- | --- |
| `ok` | `true` | Link creation succeeded; payment still requires payer action |
| `rail` | `'fiat'` | Card payment rail |
| `merchant_id` | string | Merchant derived from the trusted link |
| `checkout_id` | string | Opaque hosted Checkout handle |
| `url` | string | Hosted payer URL validated against the trusted origin and Checkout handle |
| `currency` | string | Currency returned by the service and checked against the manifest |
| `expires_at` | number | Session expiration value returned by the service |
| `product_key` | optional string | Selected Product key in Product mode |
| `human_action_required` | `true` | The payer must complete payment on the hosted page |

The SDK assembles this result: it adds `ok`, `rail`, `merchant_id`,
`product_key`, and `human_action_required` locally. The table describes the SDK
return value rather than a complete raw HTTP response schema. No amount is
included in this SDK result.

The two Card creation interfaces use different options and result names:

| Interface | Amount and currency options | Authentication | Checkout result fields |
| --- | --- | --- | --- |
| PayKit `createFiatCheckoutLink()` | Custom `amount`, or server-priced `productKey`; currency from manifest | None; public endpoint. The PayKit URL supplies the trusted origin and merchant ID | `checkout_id`, `expires_at` |
| Server SDK `createCheckoutSession()` | `paymentRails: ['fiat']`, `fiatAmount`, `fiatCurrency` | Merchant HMAC | `checkoutId`, `expiresAt` |

CLI commands:

```bash
npx goatflow-paykit inspect <quickpay-url>
npx goatflow-paykit pay-x402 <quickpay-url> --amount 10 --token USDC --chain 2345
npx goatflow-paykit pay-product <quickpay-url> --product mug --token USDC --chain 2345
npx goatflow-paykit pay-mpp <quickpay-url> --route GET:api:data
npx goatflow-paykit create-card-checkout <quickpay-url> --product mug
npx goatflow-paykit create-card-checkout <quickpay-url> --amount 10 --memo donation
```

The card command (library method `createFiatCheckoutLink`) requires an
advertised fiat rail and exactly one of a product key or custom amount.
Product prices are server-owned; product mode rejects a custom memo. The
result is a same-origin hosted checkout URL with `human_action_required: true`,
not a payment confirmation. A human enters card details on the hosted page.

QuickPay session terminal states are `PAYMENT_CONFIRMED`, `EXPIRED`, `FAILED`,
and `CANCELLED`; this is distinct from the Server SDK order model. Session
polling applies `pollTimeoutMs` as a hard cap, retains known transaction hashes
across transient status errors, can adopt a server-confirmed replacement hash
for a fresh payment, and performs five bounded grace polls when a known
transaction is reported `EXPIRED`. A forced reused session does not replace its
local hash with an unrelated prior server hash. Reconcile by session ID and
transaction hash rather than rebroadcasting after an ambiguous failure.

### Portal configuration endpoints

These use merchant portal authentication, not the HMAC server SDK. In the
reviewed Core routes, owners and members can read configuration and list
Products; mutations require the owner role. Portal page visibility is separate
from these API read permissions. Confirm the target deployment's contract.

| Action | Endpoint |
| --- | --- |
| Read/update QuickPay configuration | `GET/PUT /merchant/v1/quickpay` |
| List/create Products | `GET/POST /merchant/v1/quickpay/products` |
| Update/delete a Product | `PUT/DELETE /merchant/v1/quickpay/products/{product_key}` |

QuickPay configuration includes `quickpay_enabled`, `display_name`,
`description`, `logo_url`, `memo_required`, `max_amount_wei`,
`max_open_sessions`, `daily_session_limit`, and `max_open_sessions_merchant_wide`.

Public Product fields are `product_key` (matching `^[A-Za-z0-9._:~-]{1,64}$`, excluding `.` and `..`),
`name`, optional `description`/HTTPS `image_url`, and a positive decimal `price`.
The portal additionally exposes `enabled` and `sort_order`; the current public
manifest/client do not export or validate those fields. The product key is the
update/delete path identifier. For portal setup, see
[QuickPay and Products](merchant-guide.md#12-quickpay-and-products).

## 11. GOAT Flow MPP integration endpoints

The GOAT Flow profile's discovery, challenge, transfer, verification, receipt,
and recovery contracts are maintained in the [MPP guide](advanced/mpp.md).
They are separate from merchant HMAC APIs and do not establish standard MPP
interoperability.

## 12. Browser `PaymentHelper`

`PaymentHelper.pay(order)`:

1. Creates an `ERC20Token` for `order.tokenContract`.
2. Reads the connected signer's balance.
3. Returns failure if balance is below `order.amountWei`.
4. Sends `transfer(order.payToAddress, amount)`.
5. Waits for a receipt with status `1`.
6. Returns `{ success: true, txHash }` or `{ success: false, error }`.

It does not validate:

- wallet chain
- payer address
- order expiry
- backend order status

Applications must perform those checks.

The helper converts every `tx.wait()` exception into a failed `PaymentResult`.
Unlike the lower-level `ERC20Token` helpers, it does not classify
`TRANSACTION_REPLACED`; a successful wallet speed-up can therefore be reported
as a failure. Do not submit another transfer solely from that result. Reconcile
the original/replacement transaction and the backend order first.

## 13. Error model

TypeScript API failures throw the runtime-exported `GoatFlowError`, so
`instanceof GoatFlowError` is supported. The error preserves optional `code`
and `status`; authenticated-request failures also preserve the raw
`responseBody`. Fetch/network failures may remain native errors.

Go returns `*APIError` for non-success HTTP responses, preserving status,
optional code, and raw body; transport and JSON errors use wrapped Go errors.
The TypeScript authenticated helper accepts any `2xx` response, while Go
accepts exactly `200`. Both accept `402` only for order creation.

Browser order payment errors are returned in `PaymentResult`. MPP methods throw
`MPPError` with stable `code`, optional `httpStatus`, original `cause`, and
optional recovery context.

`onPhase` is application code and executes outside parts of the MPP error
wrapper. If that callback throws, its arbitrary error can replace the expected
`MPPError`, including during the `failed` phase. Keep it non-throwing or wrap it
locally. `bad_request` is a stable terminal MPP code for rejected HTTP `400`
verification parameters.

Do not retry a payment broadcast solely because an API call timed out. First
determine whether a transaction was already submitted.

### Identifiers and retries

| Surface | Identifier | Retry behavior |
| --- | --- | --- |
| Ordinary order create | `dappOrderId` | A duplicate is rejected rather than returning the old order. Persist `orderId` and reconcile the original intent after an ambiguous result. |
| Hosted Checkout create | `clientReferenceId` | A same-merchant duplicate conflicts rather than returning the old `checkoutId` or URL. Persist both from the successful response. |
| PayKit x402 session | `idempotencyKey` / `idempotency_key` | An existing session can be recovered and polled without automatically paying again. Each session accepts at most **32 additional retry-key aliases** beyond its original key; further new aliases return `429`. The original key and registered aliases remain eligible for valid replay. |

This alias limit applies to QuickPay x402 sessions, not the separate Card-link
or HMAC Checkout creation contract. Other limits can also return `429`; see
[QuickPay 429 troubleshooting](goat-flow-faq.md#quickpay-returned-http-429).

The TypeScript server SDK's `recoverExistingOrder` is restricted to exact,
service-recognized `topup:` retries and is not a general bypass for duplicate
orders. The public server SDKs do not provide lookup by `dappOrderId` or
`clientReferenceId`.

## 14. Runtime capability and version sources

Read enabled chains, tokens, fees, limits, and payment methods from trusted
runtime responses or operator configuration. Package installation alone does
not enable a merchant or deploy a Core/portal feature.

As verified on November 26, 2026, the package versions below are published on
npm and carry the `latest` dist-tag. These examples target those versions. Use
a lockfile to keep an application's dependency versions stable; existing
applications do not need to upgrade merely because this documentation was
corrected. The [release procedure](../RELEASING.md) governs future releases.
All four packages declare Node.js 18 or later where Node is used.

| Package | Published version | Primary use | Exact install |
| --- | ---: | --- | --- |
| [`goatflow-sdk`](../goatx402-sdk/README.md) | [0.3.0](../goatx402-sdk/CHANGELOG.md) | Browser wallet transfers and the GOAT Flow MPP adapter | `npm install goatflow-sdk@0.3.0` |
| [`goatflow-sdk-server`](../goatx402-sdk-server-ts/README.md) | [0.4.0](../goatx402-sdk-server-ts/CHANGELOG.md) | TypeScript merchant backend and HMAC-authenticated APIs | `npm install goatflow-sdk-server@0.4.0` |
| [`goatflow-checkout`](../goatx402-checkout/README.md) | [0.2.0](../goatx402-checkout/CHANGELOG.md) | Hosted Checkout browser integration | `npm install goatflow-checkout@0.2.0` |
| [`goatflow-paykit`](../goatx402-quickpay/README.md) | [0.4.0](../goatx402-quickpay/CHANGELOG.md) | PayKit payer/agent library and CLI | `npm install goatflow-paykit@0.4.0` |

### Migration from QuickPay

Replace the `goatflow-quickpay` dependency and import specifiers with
`goatflow-paykit`. The class name `QuickPayClient` remains available. Use
`npx goatflow-paykit`; the new package also installs a `goatflow-quickpay`
executable alias, but `npx goatflow-quickpay` without that local installation
still resolves the old npm package, not PayKit.

Prefer `PAYKIT_PRIVATE_KEY` and `PAYKIT_RPC[_<chainId>]`; the corresponding
`QUICKPAY_*` environment variables remain supported. Public merchant links
accept `/paykit/{merchant_id}` and legacy `/quickpay/{merchant_id}`, with the
same optional `/agent.md` or `/manifest.json` suffix. The client continues to
derive API endpoints under `/quickpay/v1` on the trusted origin. Repository
directory names remain `goatx402-*`.

The TypeScript MPP middleware package name
`@goatnetwork/mpp-middleware`, the Go modules, contracts, and demo are not
public npm packages. Follow their package README files for source-based use;
do not infer Registry availability from a local `package.json`.

### Published SDK boundary

As checked on November 26, 2026, `goatflow-sdk-server@0.4.0` and the public Go
server SDK source do not expose `cryptoSubmissionProtocol` /
`CryptoSubmissionProtocol` or a `renewCheckoutSession()` /
`RenewCheckoutSession()` method. `goatflow-checkout@0.2.0` reports hosted-page
errors as `checkout_error`; it does not expose a distinct
`checkout_superseded` reason.

These guides target those public interfaces. Do not copy newer Core-only
options into these examples or assume that installing the current packages
enables submission arbitration or checkout renewal. Those capabilities require
a compatible server SDK, Core, and hosted page verified together. A failed or
expired attempt is not permission to create a replacement payable session.

The public Go server SDK requires Go 1.25. Its Card fields and merchant token
mapping limitations are documented in [the Go README](../goatx402-sdk-server-go/README.md#other-operations).
Polling differences are documented in [Read order status](#5-read-order-status).

The canonical repository is [GOATNetwork/goat-flow](https://github.com/GOATNetwork/goat-flow).
The former `GOATNetwork/x402` URL redirects to it; published package metadata
still records the former name. Older generated examples may mention
`pay.goat.network`; use the [current origins](#1-origins-and-configuration).

These package/source checks do not establish the Core or hosted-page revision
running on Mainnet or Testnet3. Checkout redirect settings in particular depend
on the target Core and portal deployment.

## Appendix A: Operator-provisioned callback compatibility

This appendix is not part of public DIRECT onboarding. Use it only when the
target merchant and environment have an explicit operator deployment contract.

Checkout Session compatibility supports either a decimal `price`, or the
legacy fixed-wei combination of `chain_id`, `fixed_amount_wei`, and
`acceptable_tokens`. For these explicitly provisioned sessions,
`checkout_type` is `DELEGATE`. The additional server-SDK field mapping is:

| Wire field | TypeScript | Go | Use |
| --- | --- | --- | --- |
| `chain_id` | `chainId` | `ChainID` | Legacy fixed-wei source chain |
| `fixed_amount_wei` | `fixedAmountWei` | `FixedAmountWei` | Legacy fixed-wei amount |
| `acceptable_tokens` | `acceptableTokens` | `AcceptableTokens` | JSON-stringified token addresses |
| `callback_calldata` | `callbackCalldata` | `CallbackCalldata` | Optional create-time callback calldata |

Only create-time `callback_calldata` in the legacy fixed-wei form is
server-authoritative and guarantees exact callback bytes. The decimal-price
form rejects create-time calldata. A
`public_metadata_json.callback_template` value is only a hosted-UI encoding
hint: bind-time calldata is buyer-controlled, may be omitted or replaced, and
is not revalidated against the template. The callback contract must enforce its
own selector, parameter, and permission policy.

The TypeScript `createDelegateCheckoutSession()` and Go
`CreateDelegateCheckoutSession()` helpers remain deprecated compatibility
wrappers around `createCheckoutSession()` / `CreateCheckoutSession()`.

An order-create request may include `callback_calldata`. The authoritative
challenge can then use `ERC20_3009` or `ERC20_APPROVE_XFER`, return an
operator-provisioned recipient, and include
`extensions.goatx402.signatureEndpoint` and `calldata_sign_request`.

The signature request supplies the complete EIP-712 `domain`, `types`,
`primaryType`, and `message`. `primaryType` is `Eip3009CallbackData` or
`Permit2CallbackData`; sign the returned structure without rebuilding or
selectively copying it. Its `domain.chainId` can differ from the transfer
source chain, and the SDK does not switch either chain for the application.

After the buyer signs, send the signature to the merchant backend. The backend
submits it with `submitCalldataSignature(orderId, signature)` or
`SubmitCalldataSignature(...)`:

```http
POST /api/v1/orders/{order_id}/calldata-signature
Content-Type: application/json

{ "signature": "0x..." }
```

Keep merchant HMAC credentials on the backend.
