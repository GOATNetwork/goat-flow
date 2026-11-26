# GOAT Flow

GOAT Flow provides hosted checkout, merchant APIs, and payer tools for crypto
and eligible Card payments. This repository contains the public SDKs and
examples. For crypto DIRECT payments, the buyer transfers ERC-20 tokens to the
merchant's receiving address. Card payers complete the hosted provider flow.
Available payment methods depend on the deployment and merchant configuration.

**For hosted crypto checkout on Testnet3 using TypeScript, start with the
[Developer Quick Start](docs/goat-flow-developer-quickstart.md).** It follows
one path: create a payment session on your backend, open hosted checkout, and
confirm payment before fulfillment.

| Task | Documentation |
| --- | --- |
| Set up a merchant or manage payments | [Merchant Guide](docs/merchant-guide.md) |
| Find an endpoint, field, environment, or SDK limitation | [API Reference](docs/goat-flow-api-reference.md) |
| Diagnose a failed integration | [Troubleshooting](docs/goat-flow-faq.md) |
| Choose a different integration | [Documentation index](docs/README.md) |

## Packages

| Module | Use |
| --- | --- |
| [goatflow-checkout](goatx402-checkout/README.md) | Open hosted checkout in a browser |
| [goatflow-sdk-server](goatx402-sdk-server-ts/README.md) | TypeScript merchant backend |
| [Go server SDK](goatx402-sdk-server-go/README.md) | Go merchant backend; check its documented limitations |
| [goatflow-sdk](goatx402-sdk/README.md) | Custom EVM wallet UI and GOAT Flow MPP adapter |
| [goatflow-paykit](goatx402-quickpay/README.md) | Agent and CLI payer tools; replaces goatflow-quickpay |
| MPP middleware: [TypeScript](goatx402-mpp-middleware-ts/README.md), [Go](goatx402-mpp-middleware-go/README.md) | Verify GOAT Flow profile receipts on a resource server |
| [Demo](goatx402-demo/README.md) | Local integration examples |
| [Contracts](goatx402-contract/README.md) | Optional callback contracts and local test tokens |

Published versions, runtime requirements, and migration notes have one home:
[package compatibility](docs/goat-flow-api-reference.md#14-runtime-capability-and-version-sources).
Repository directory names remain `goatx402-*`. Package releases follow
[RELEASING.md](RELEASING.md).

## Integration boundaries

- Read enabled chains, tokens, prices, and receiving addresses from the target
  environment. A static chain list is not a merchant's available configuration.
- Keep API secrets on your backend. Confirm payment using trusted server data;
  a browser callback, redirect, or transaction hash alone cannot authorize fulfillment.
- DIRECT checkout does not require deploying a callback contract. The optional
  [callback contracts](goatx402-contract/MERCHANT_CALLBACK.md#calldata-execution-semantics)
  require a separate operator-provisioned integration and contract review.
- [MPP](https://mpp.dev/overview) is an independent open protocol. This
  repository's [MPP adapter](docs/advanced/mpp.md) implements GOAT Flow's JSON
  endpoints and signed-receipt profile; generic MPP interoperability is not established.

## License

No single repository-wide license has been declared. The four release-managed
npm packages each include their own MIT `LICENSE` file. Review the relevant
module's license before reuse.
