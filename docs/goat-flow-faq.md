# Troubleshooting

Start with the symptom below. For the normal flow, use the
[Developer Quick Start](goat-flow-developer-quickstart.md).

## Order creation returned HTTP 402

For `POST /api/v1/orders`, `402` is the successful payment challenge; both
server SDKs normalize it. A `402` from status, proof, cancellation, or session
creation is an error. See the [endpoint table](goat-flow-api-reference.md#3-merchant-endpoint-summary).

## Checkout creation returned 400 or a return URL was rejected

Check the response code/body, then the session fields and merchant eligibility.
If you supplied `successUrl` or `cancelUrl`, configure the merchant's redirect
allowlist first. An empty list rejects both; a path prefix must match itself or
a child path. See [Checkout redirects](merchant-guide.md#checkout-return-urls).

## A duplicate ID returned a conflict

Ordinary order creation and hosted-session creation do not replay the original
result. Use the saved order/session ID to reconcile. A timeout does not prove
creation failed; do not allocate another ID merely to bypass the conflict.
PayKit's idempotency keys have a different contract. See
[Identifiers and retries](goat-flow-api-reference.md#identifiers-and-retries).

## QuickPay returned HTTP 429

For x402 session creation, inspect the response error. If it says
`session retry key limit reached; reuse an existing idempotency key`, stop
generating new keys and reuse the original key or an already registered alias
for the same payment intent. The session permits 32 additional aliases beyond
its original key; see [Identifiers and retries](goat-flow-api-reference.md#identifiers-and-retries).

Other `429` responses can indicate request-rate limits, too many open sessions,
or a merchant's daily/session-capacity limit. Follow the reported limit instead
of assuming every `429` is an alias-limit error or creating another payment.

## The checkout window did not open

Popup and tab modes need `open()` synchronously inside the click handler.
Calling it after awaiting session creation can trigger a popup blocker. Use
`redirectToCheckout()` for an asynchronous flow. If a strict COOP policy breaks
the opener channel, use redirect mode. See the [Checkout SDK](../goatx402-checkout/README.md).

## Checkout shows success but the order is not fulfilled

A browser success callback or redirect is a UI signal. On your backend, use the
saved session to resolve the payment, verify its trusted status and expected
terms, and fulfill once. Crypto orders may advance directly to `INVOICED`
between polls. Card uses its own confirmation contract and need not have a
transaction hash. See [confirmation](goat-flow-developer-quickstart.md#5-read-the-payment-result-on-the-backend).

## The wallet sent a transaction but confirmation timed out

Retain the transaction hash and original payment intent. Check the chain
receipt and trusted order/session status before any new transfer. A replaced
transaction or delayed watcher can require reconciliation.
`PaymentHelper.pay()` does not classify `TRANSACTION_REPLACED`. For MPP, resume
[verification with the saved challenge and hash](advanced/mpp.md#7-recovery--never-pay-twice).

## Go polling timed out, even with invalid credentials

The current Go waiter retries every status-read error, including `401`, `403`,
and `404`. Use explicit `GetOrderStatus` polling when those errors must surface
immediately. Pass a context deadline to bound in-flight requests; the waiter's
timeout alone is checked at interval ticks. See [polling behavior](goat-flow-api-reference.md#5-read-order-status).

## Go returned an empty supported-token list

The current Go type expects `supported_tokens`; Core can return `wallets[]`.
An empty `SupportedTokens` slice does not establish that no tokens are configured.
Check the wire response or decode that field separately. See
[Read merchant](goat-flow-api-reference.md#8-read-merchant).

## Card fields, renewal, or submission protocol options are missing

Backend features and installed SDK capabilities are separate. The public Go
helper lacks Card fields; current public server SDKs lack protocol 1 and session
renewal. Do not copy Core-only options into a published SDK example. Check the
[compatibility baseline](goat-flow-api-reference.md#14-runtime-capability-and-version-sources).

## A chain, token, or payment method is unavailable

Read the target merchant's current portal/discovery configuration. Testnet3
and Mainnet have separate accounts, credentials, receiving addresses, fees, and
capabilities. Crypto hosted sessions require QuickPay; authenticated Card-only
sessions instead require the deployment's fiat/Connect enablement. Public
PayKit Card-link creation also requires QuickPay. See
[session requirements](goat-flow-api-reference.md#9-hosted-checkout-sessions).

## Which webhook event and signature should I implement?

Use the contract confirmed for your deployment, with a redacted real delivery
fixture. The public SDKs do not provide a deployment-approved production
webhook schema. Do not invent an event ID or infer signature/replay rules from
an example. Follow [webhook setup and verification](merchant-guide.md#9-webhook-configuration).

## Getting help

Contact [Support@goat.network](mailto:Support@goat.network) with the environment,
SDK version, operation, timestamp, and sanitized error code/status. Keep API
secrets, webhook secrets, wallet keys, and opaque checkout handles out of tickets.
