# Custom wallet integration

Use this path when your application owns the wallet UI. For the hosted page,
follow the [Developer Quick Start](../goat-flow-developer-quickstart.md).
This guide covers DIRECT ERC-20 transfers and requires an approved merchant,
configured receiving tokens, and a buyer wallet funded with tokens and native gas.

Install `goatflow-sdk-server` on the backend. In the browser application,
install `goatflow-sdk` and `ethers`, which the wallet example imports:

```bash
npm install goatflow-sdk-server@0.4.0
npm install goatflow-sdk@0.3.0 ethers@^6
```

The snippets show SDK integration points; your application supplies authenticated
routes, storage, and fulfillment. Use an EIP-1193 wallet/provider. No tested
minimum-browser-version matrix is published; the SDK uses modern browser
features including `BigInt`, `fetch`, `URL`, and Web Crypto.

## 1. Configure the backend

```bash
GOATX402_API_URL=https://flow-api.testnet3.goat.network
GOATX402_API_KEY=your_api_key
GOATX402_API_SECRET=your_api_secret
```

Keep API credentials on the backend. The API origin itself is public.

## 2. Create and map the order

The sample chain, symbol, and amount must match your target merchant configuration.
Read the price from your own backend and validate the buyer's selected chain/token.
Persist the returned GOAT Flow order ID against the authenticated buyer's business order.

The server and browser SDKs intentionally expose different `Order` shapes.
Map the object explicitly before returning it to the frontend.

```ts
import {
  GoatFlowClient,
  type Order as ServerOrder,
} from 'goatflow-sdk-server'

const client = new GoatFlowClient({
  baseUrl: process.env.GOATX402_API_URL!,
  apiKey: process.env.GOATX402_API_KEY!,
  apiSecret: process.env.GOATX402_API_SECRET!,
})

function toClientOrder(order: ServerOrder, fromAddress: string) {
  return {
    orderId: order.orderId,
    flow: order.flow,
    tokenSymbol: order.tokenSymbol,
    tokenContract: order.tokenContract,
    fromAddress,
    payToAddress: order.payToAddress,
    chainId: order.fromChainId,
    amountWei: order.amountWei,
    expiresAt: order.expiresAt,
    calldataSignRequest: order.calldataSignRequest,
  }
}

export async function createOrder(
  dappOrderId: string,
  fromAddress: string,
) {
  const order = await client.createOrder({
    dappOrderId,
    chainId: 48816,
    tokenSymbol: 'USDC',
    fromAddress,
    amountWei: '10000000', // Example only: 10 tokens when configured decimals = 6.
  })

  return toClientOrder(order, fromAddress)
}
```

Generate `dappOrderId` once for the cart or payment intent, persist it before
the request, and reuse the same value when reconciling an ambiguous attempt. An
ordinary duplicate order create is rejected; it does not return the earlier
order. Persist a successful `orderId`, and after a timeout or duplicate error
reconcile that original order instead of generating a new ID and potentially
requesting a second payment. Do not derive the ID from the current timestamp
inside `createOrder()`.

The TypeScript SDK's `recoverExistingOrder` option is restricted to exact retries of
deterministic `topup:` orders accepted by the service. It is not a general
merchant idempotency switch; do not manufacture that prefix to bypass duplicate
rejection.

Under the hood, successful order creation returns HTTP `402 Payment Required`.
The server SDK treats it as success and normalizes the x402 body.

The server SDK accepts `402` as success only for order creation. An unexpected
`402` from status, proof, checkout, signature, or cancellation fails closed.

## 3. Validate and pay in the browser

```ts
import { PaymentHelper, type Order } from 'goatflow-sdk'
import { ethers } from 'ethers'

export async function payOrder(
  order: Order,
  walletProvider: ethers.Eip1193Provider,
): Promise<string> {
  const provider = new ethers.BrowserProvider(walletProvider)
  const signer = await provider.getSigner()

  const network = await provider.getNetwork()
  if (Number(network.chainId) !== order.chainId) {
    throw new Error(`Switch wallet to chain ${order.chainId}`)
  }

  const payer = await signer.getAddress()
  if (payer.toLowerCase() !== order.fromAddress.toLowerCase()) {
    throw new Error('Connected wallet does not match the order payer')
  }

  if (Math.floor(Date.now() / 1000) >= order.expiresAt) {
    throw new Error('Order expired')
  }

  const payment = new PaymentHelper(signer)

  if (order.flow !== 'ERC20_DIRECT' || order.calldataSignRequest) {
    throw new Error(
      'This DIRECT quick start does not handle operator-provisioned callback orders',
    )
  }

  const result = await payment.pay(order)
  if (!result.success || !result.txHash) {
    throw new Error(result.error ?? 'Payment failed')
  }
  return result.txHash
}
```

Pass the EIP-1193 provider selected by your wallet connector as `walletProvider`.
`PaymentHelper.pay()` does not classify `TRANSACTION_REPLACED`. If a wallet
speed-up is reported as failed, reconcile the original/replacement transaction
and backend order before considering another transfer.

Callback signing is an operator-provisioned compatibility path, not part of
this DIRECT quick start. If the target deployment explicitly requires it, use
the complete field, callback-chain, and signature-submission contract in the
[callback compatibility contract](../goat-flow-api-reference.md#appendix-a-operator-provisioned-callback-compatibility).

`PaymentHelper.pay()` checks token balance, submits the ERC-20 transfer, and
waits for a successful receipt. It returns failures in `PaymentResult`; it does
not check chain, payer, or expiration.

## 4. Confirm on the backend

Do not fulfill only because the wallet transaction returned successfully.

```ts
const orderStatus = await client.getOrderStatus(orderId)

const fulfillable =
  orderStatus.status === 'PAYMENT_CONFIRMED' ||
  orderStatus.status === 'INVOICED'

if (fulfillable) {
  const proof = await client.getOrderProof(orderId)
  // Compare the order and proof with your saved business payment terms.
  // Then apply your application's atomic, idempotent fulfillment transition.
}
```

Current SDK status values are:

- `CHECKOUT_VERIFIED`
- `PAYMENT_CONFIRMED`
- `INVOICED`
- `FAILED`
- `EXPIRED`
- `CANCELLED`

Server SDK order waiters treat `PAYMENT_CONFIRMED` and `INVOICED` as successful
terminal states. Core can advance a DIRECT order from `PAYMENT_CONFIRMED` to
`INVOICED` in one watcher transaction, so a poller may observe only
`INVOICED`. Before fulfillment, still validate the authenticated order's
merchant context, chain, token, amount, recipient, and transaction identity.

Cancel an abandoned order only while it remains `CHECKOUT_VERIFIED`:

```ts
await client.cancelOrder(orderId)
```

## ERC-20 approval helpers

Order payment itself uses `transfer()` and does not require an allowance.
Approval helpers are for other integration needs.

```ts
const exactTx = await payment.approveToken(token, spender, amount)
const unlimitedTx = await payment.approveToken(
  token,
  spender,
  amount,
  { unlimited: true },
)
```

The helper:

- validates `bigint` and uint256 bounds before any transaction
- skips writes when the target allowance is already set
- uses a direct-write `eth_call` probe
- falls back to confirmed `approve(0)` only when required
- follows matching fee-bump replacements

Use `ERC20Token.setApproval()` when the reset transaction hash is also needed.

## Reference

- [Order fields and raw x402 response](../goat-flow-api-reference.md#4-create-order)
- [TypeScript and Go polling differences](../goat-flow-api-reference.md#5-read-order-status)
- [Go backend integration](../../goatx402-sdk-server-go/README.md)
- [Troubleshooting](../goat-flow-faq.md)
