# Merchant Guide

Use this guide for Merchant Portal setup and operations. Developers should
start with the [Developer Quick Start](goat-flow-developer-quickstart.md).
Screenshots include Mainnet and Testnet3 examples from earlier portal versions.
Navigation names below follow the portal source reviewed on November 26, 2026;
confirm which version your deployment runs. Use the [environment table](goat-flow-api-reference.md#1-origins-and-configuration)
to open the correct portal, and keep test and production configuration separate.

## First-time setup

| Step | Action | Complete when |
| --- | --- | --- |
| 1 | [Register and sign in](#3-register-a-merchant-account) | Merchant approved and enabled |
| 2 | [Configure receiving addresses](#6-configure-receiving-addresses), or confirm Card eligibility with the operator | The intended payment method is available |
| 3 | [Configure QuickPay / Products](#12-quickpay-and-products) and [API keys](#8-api-keys-management) where needed | Your link or backend can create the intended checkout |
| 4 | Complete the chosen [application integration](goat-flow-developer-quickstart.md) | A test buyer can pay and the backend confirms it |
| 5 | Complete [launch checks](#launch-checks) | Fulfillment, reconciliation, and operational ownership are verified |

For daily operations: [orders](#11-order-management) ·
[fees and top-up](#13-balance-fees-and-top-up) ·
[team](#10-team-management-and-invite-codes) · [security](#4-approval-login-and-account-security) ·
[webhooks](#9-webhook-configuration) · [audit logs](#14-audit-logs).

## 1. Overview

| Portal group | Pages |
| --- | --- |
| Overview | Dashboard |
| Payments | All Orders · Refund Requests · Disputes · Reconciliation |
| Payment Setup | Payment Guide · Crypto Payments · Card Payments (Stripe) · PayKit Settings |
| QuickPay | Accept Any Amount · Sell Products |
| Developer | API & Checkout Settings · MPP Routes |
| GOAT Flow Fees | Fee Balance · Top Up Fee Balance |
| Account & Security | Profile · Team · Security · Audit Logs |

The table shows DIRECT merchant navigation; owner-only pages are hidden from
members. Older screenshots use **Receiving Tokens & Addresses** for Crypto
Payments, **Hosted Checkout (QuickPay)** for the public-link setup, **QuickPay
Products** for Sell Products, and **Paid API Routes (MPP)** for MPP Routes.

## 2. Payment Mode and Available Assets

For crypto DIRECT payments, the buyer sends ERC-20 tokens to your configured
receiving address on the selected chain. Configure a receiving address for
each accepted chain/token pair. The buyer needs the token plus native gas.

Card payers complete the hosted provider flow without a crypto wallet.
Merchant fiat eligibility, currency, connected Stripe account, and provider
test/live mode must be confirmed in the target deployment. Crypto receiving
setup does not enable Card payments.

Read available tokens, contracts, decimals, amount limits, and fees from the
active portal/API configuration. Testnet3 settings do not establish Mainnet
availability. A browser success signal does not itself confirm payment.

## 3. Register a Merchant Account

### 3.1 Open the Registration Page

Visit the Merchant Portal and select **Apply** to open the registration form.

![Mainnet Registration Page](./images/68-mainnet-register-empty.png)

### 3.2 Fill in Registration Details

![Completed Mainnet Registration Form](./images/69-mainnet-register-filled.png)

The registration form includes:

| Field | Description | Format Requirements |
| --- | --- | --- |
| **Merchant ID** | Unique merchant identifier; cannot be changed after registration | At most 64 ASCII letters, digits, hyphens, or underscores. Reserved IDs and `topup-` / `topup_` prefixes are not available. |
| **Business name** | Display name; can be changed later subject to name validation and availability | After normalization: at least 2 visible characters (grapheme clusters), at most 128 Unicode code points, and no control or invisible Unicode characters. |
| **Work email** | Owner login email | Valid email address, at most 256 Unicode code points |
| **Password** | Owner login password | 8 to 72 UTF-8 bytes; a non-ASCII character can use several bytes. Set up 2FA after signing in. |

Business names are normalized for Unicode and whitespace and checked for
uniqueness across merchants using a case-folded comparison. For self-service
registration, names held by pending or approved applications remain reserved
while held in those states; rejected names remain reserved for 30 days after
rejection. A conflicting name returns HTTP `409`.

Self-service registration supports **DIRECT only**. The API defaults an omitted
`receive_type` to `DIRECT` and rejects other modes.

### 3.3 Submit Application

Click **Submit application** when done.

The registration endpoint (`POST /merchant/v1/auth/register`) defaults to
**5 requests per IP per hour**; IPv6 addresses in the same `/64` share a limit.
Exceeding the limit returns HTTP `429`. The limit is deployment-configurable;
wait before retrying instead of repeatedly submitting the form.

After submission, the portal displays:

> Registration submitted. Your account will be available after admin approval.
> No access token or session is issued until then.

![Mainnet Registration Pending Approval](./images/71-mainnet-registration-pending.png)

The applicant is not signed in and no access or refresh session is issued until
the application is approved. The GOAT Flow operations team performs the approval
and activates the owner account.

---

## 4. Approval, Login, and Account Security

### 4.1 Admin Approval

After registration, a GOAT Flow administrator reviews the application. Approval
enables the merchant account. QuickPay and MPP are separate capabilities;
inspect **Profile** or the relevant setup page to see whether they are enabled
for the account.

### 4.2 Login

The portal provides **Sign in**, **Apply**, and **Use invite** tabs.

![Mainnet Login Page](./images/70-mainnet-login.png)

Once approved, enter the work email and password used during registration. If
2FA is enabled, complete the TOTP or recovery-code challenge before entering the
dashboard.

If login fails, the generic `invalid email or password` response does not
distinguish an invalid credential from an activation or account-state problem.
If the credentials are correct but access still fails, contact
[Support@goat.network](mailto:Support@goat.network) to confirm the account state
or request a reset.

### 4.3 Change Password

Every authenticated owner or member can open **Account & Security → Security**
and select **Change password**. The form requires:

- current password
- new password
- confirmation of the new password

The new password must be 8 to 72 UTF-8 bytes and must differ from the current
password.

![Change Password](./images/44-change-password.png)

Lockout thresholds and administrator reset policies can vary. Contact support if
self-service recovery is unavailable.

### 4.4 Self-Service 2FA

2FA is **per user and not owner-gated**. Enroll, confirm, or disable TOTP on the
**Account & Security → Security** page. The account is password-only until 2FA
is enabled; after that, every sign-in requires a 6-digit authenticator code.

![Account Security and 2FA](./images/43-security-current.png)

Store recovery codes securely. They are meant for account recovery if the authenticator is unavailable.

Do not publish screenshots of the enrollment QR code, shared secret, or recovery
codes.

### 4.5 Lost Password or Authenticator

If you lose your password or 2FA authenticator and
cannot recover with a one-time recovery code, contact the GOAT Flow operations
team at [Support@goat.network](mailto:Support@goat.network).

### 4.6 Sessions

Portal sessions use short-lived access tokens with refresh sessions. After a
password or 2FA change, sign out other sessions where possible and sign in again.
Do not rely on a fixed session-expiry or invalidation delay.

---

## 5. Dashboard Overview

After login, the owner lands on the Dashboard, which shows order counts,
revenue summaries, and recent orders.

![New Merchant Dashboard](./images/49-new-merchant-dashboard.png)

For incomplete DIRECT setup, select **Resume setup** to open
**Payment Setup → Payment Guide** (`/acceptance/onboarding`). Confirm DIRECT mode, add receiving rows,
review optional API keys, and review the optional self-test. API keys are
required when your backend creates orders/sessions, but not for a public
Product link. The self-test is guidance, not an approval gate.

![Post-Approval Onboarding](./images/50-portal-onboarding.png)

## 6. Configure Receiving Addresses

Open **Payment Setup → Crypto Payments** (`/acceptance/receiving`).

In DIRECT, the buyer wallet sends tokens to the merchant
wallet address returned for the order on the selected EVM chain.

### 6.1 Add a Receiving Address

Click **Add address** in the top right to open the form. Token options come from
the supported token list, filtered to EVM and already-configured pairs.

![Add Receiving Address](./images/37-add-address-current.png)

The **DIRECT receiving addresses** table lists one row per accepted pair:

| Column | Description |
| --- | --- |
| **Chain** | One of the EVM chains offered by the active deployment |
| **Symbol** | Token symbol, such as USDC or USDT |
| **Token contract** | The token's ERC-20 contract address |
| **Address** | Your EVM receiving address (`0x` + 40 hex characters) |

The form enforces these rules:

- Each Chain + Token combination can only have one address.
- Address and token contract must be valid EVM addresses.
- Available chains/tokens come from deployment configuration.
- The onboarding picker initially selects all available test rows; review the
  selection before confirming so that one address is not unintentionally assigned
  to every chain and token.

To configure several pairs at once, use the **EVM setup draft** panel: add rows,
then select **Submit**. Each row is submitted independently and reports its own
success or error state; review every row before continuing.

![Receiving Addresses List](./images/53-new-merchant-receiving.png)

---

## 7. Merchant Settings

Open **Account & Security → Profile** to view your merchant identity and edit
business details. Receive type and merchant ID are read-only.

![Merchant Profile](./images/61-new-merchant-profile.png)

### Identity and capability flags

The account card shows fixed identity and capability fields:

| Field | Editable | Description |
| --- | --- | --- |
| Merchant ID | No | Set during registration |
| Receive type | No | `DIRECT`; fixed at registration |
| Status | No | Enabled / Disabled |
| Hosted Checkout (QuickPay) | No | Whether QuickPay is enabled for this merchant |
| MPP enabled | No | Whether Paid API Routes (MPP) are enabled |
| Role | No | Owner or Member |

### Business details

| Field | Editable | Description |
| --- | --- | --- |
| Business name | Yes | Shown to buyers on your hosted checkout and on receipts |
| Logo URL | Yes | Optional; must be an `https://` image URL without embedded credentials |

Click **Save changes** after modifications.

### Per-chain payment limits

The **Per-chain payment limits** panel is **read-only**.
These are merchant-level per-chain limits set by GOAT Flow risk controls and
shown for reference (for example `max_frozen_amount_usd`,
`max_pending_orders`).

---

## 8. API Keys Management

Open **Developer → API & Checkout Settings** (`/developer`). The page heading
is **API Keys, Webhooks & Checkout**; older versions may show
**Programmatic API & Webhooks**.

Your backend needs API credentials to create orders or Checkout Sessions,
including Card-only sessions. Public Product links and public PayKit Card-link
creation do not need merchant API credentials in the buyer page.

An owner selects **Create API Keys** initially or **Rotate API Keys** to replace
existing credentials. Save the
secret immediately in your backend secret store: it is displayed once and
cannot be retrieved later. After suspected exposure, rotate the credentials
and confirm the old ones no longer work.

Give developers the [backend setup instructions](goat-flow-developer-quickstart.md#2-install-and-configure).
Keep secrets out of browser code, screenshots, tickets, and repositories.

### Checkout return URLs

When the target Core and portal support Checkout settings, an owner can:

1. Open **Developer → API & Checkout Settings**, then **Checkout redirects**
   at `/developer#checkout-redirects`.
2. Add one allowed HTTPS origin or path prefix per line, such as
   `https://shop.example.com/checkout`, then save and reload.
3. Supply matching `successUrl` / `cancelUrl` values when creating sessions,
   such as `https://shop.example.com/checkout/success`.

An empty allowlist rejects new sessions containing return URLs. Omitting both
URLs avoids redirect validation. The setting does not enable QuickPay or Card
payments. If the section is absent, confirm deployment support with the operator.
Exact matching, limits, and existing-session behavior are in the
[API Reference](goat-flow-api-reference.md#checkout-redirect-settings).

## 9. Webhook Configuration

On **API & Checkout Settings**, select **Add webhook**. The portal accepts up to three HTTPS
destinations per merchant; localhost is not allowed. Select events offered by
your deployment, then save the one-time webhook secret in your backend.

![Add Webhook](./images/40-add-webhook-current.png)

The portal options and Core subscription allowlist reviewed on November 26,
2026 contain these nine events:

- `order.invoiced`
- `quickpay.payment.confirmed`
- `quickpay.checkout.completed`
- `fiat.payment.succeeded`
- `fiat.payment.refunded`
- `fiat.refund_request.updated`
- `fiat.dispute.action_required`
- `fiat.dispute.updated`
- `fiat.dispute.closed`

Event availability and payloads can differ by deployment; the installed public
SDK does not define the deployment's webhook contract.

The public repository does not provide a deployment-approved production
webhook schema or delivery fixture. Before implementing a receiver, obtain a
redacted real delivery and confirm the event schema, signature input/headers,
timestamp/replay rules, and retry behavior with the deployment owner. Preserve
raw request bytes if the confirmed signing contract requires them.

### Verify before fulfillment

1. Authenticate the notification and apply its replay rules before trusting fields.
2. Resolve its verified order/session identity to your saved business order.
   A public `client_reference_id` alone does not establish ownership.
3. Check the successful state and expected payment terms: chain, token, recipient,
   and amount for Crypto; currency, amount, and provider mode for Card.
   A Card confirmation need not have a transaction hash or crypto proof.
4. Durably enqueue valid events and acknowledge promptly. Make the final
   fulfillment update atomic and idempotent so duplicate delivery cannot ship twice.
5. Test real delivery, delay, receiver failure, retry, and reconciliation in the
   intended environment. A browser callback must not bypass server verification.

Identifiers are not interchangeable: `checkout_id` is the opaque bearer handle;
`checkout_session_id` can be an internal row ID; `order_id` is an order;
`session_id` / `quickpay_session_id` identify a PayKit session. Use the field
defined by the confirmed event contract. Do not assume `event_id` exists or that
a delivery timestamp is a stable deduplication key. Retain the confirmed
contract and sanitized fixture with your application's tests.

## 10. Team Management and Invite Codes

Open **Account & Security → Team**. Merchant owners create and revoke
**single-use member invite codes** here. Invited members can access operational
and billing views plus their own Security page, but owner-only setup pages are
hidden.

> The invitee accepts the code on the public portal's **Use invite** tab, which is
> labeled **Join workspace** — the same merchant, joined as a member.

### 10.1 Role permissions

The current portal exposes these role permissions. Confirm role enforcement in
your target environment before relying on it for sensitive operations.

| Feature | Owner | Member |
| --- | --- | --- |
| View Dashboard | Yes | Yes |
| View Orders | Yes | Yes |
| View Order Reconciliation | Yes | Yes |
| Access Fee Balance / Fee Top-up | Yes | Yes |
| Manage own password and 2FA | Yes | Yes |
| Modify profile and receiving settings | Yes | No |
| Manage API keys | Yes | No |
| Manage webhooks | Yes | No |
| Manage QuickPay and Products | Yes | No |
| Manage MPP routes | Yes | No |
| Manage Team / Invite Codes | Yes | No |
| View Audit Logs | Yes | No |

### 10.2 Create an Invite Code

Go to the **Team** page and click **Create invite**.

![Create Invite Code](./images/42-create-invite-current.png)

- Owner-created invite codes create `member` users.
- Expiration defaults to 72 hours.
- Expiration may be set up to 720 hours.
- Each invite code is single-use.

The full invite code is shown once. Do not publish it while it is active.

### 10.3 Invitee Registration

The invitee opens the Merchant Portal and uses the **Use invite** tab (Join workspace).

![Invite Registration](./images/67-use-invite-empty.png)

Fill in:

- **Invite code** - The single-use code provided by the workspace owner (starts with `inv_`)
- **Email** - The invitee's email
- **Password** - A password following the 8 to 72 UTF-8 byte rule

The invitee joins as a member. Invite-code registration does not require new merchant approval because the merchant is already approved.

After registration, the member is signed in to the same merchant workspace.
Direct navigation to an owner-only page redirects the member back to
**Security**.

![Member Dashboard](./images/62-member-dashboard-current.png)

The Team table marks a redeemed code as **Used** and records the member email.

![Used Invite Code](./images/63-team-invite-used.png)

---

## 11. Order Management

Open **Payments → All Orders** (`/payments/orders`) to view orders.

![Orders List](./images/64-new-merchant-orders.png)

### Filters

| Filter | Options |
| --- | --- |
| **Status** | All / order statuses |
| **Payment method** | All EVM flows / Direct transfer / EIP-3009 / Permit2 |
| **Chain** | All chains / configured deployment chains |
| **From / To** | Full timestamp range |

### Order List Fields

| Column | Description |
| --- | --- |
| Order ID | System order ID |
| Dapp Order | DApp-side order number |
| Payment method | Direct transfer, EIP-3009, or Permit2 |
| Type | Order type, such as x402 payment |
| Token | Payment token, such as USDC |
| Amount | Payment amount |
| User | Payer address |
| Status | Order status |
| Created | Creation time |
| View | Opens the order detail dialog |

Click **View** to see order and transfer data. The dialog includes
chains, token contract and decimals, receiving address, payer, transaction hash,
expiry, memo, timestamps, confirmation details, and compatibility fields related
to the selected method. A `Payout Tx` field in the portal does not mean GOAT Flow
pays out, releases, or settles customer funds to a DIRECT merchant; the buyer's
tokens move directly to the merchant receiving address.

![Order Detail](./images/35-order-detail.png)

The example shows a Testnet3 DIRECT transfer displayed as `INVOICED`, the
successful terminal state commonly visible after Core records the direct
transfer and invoice.

### 11.1 Order status and fulfillment

For crypto orders, `PAYMENT_CONFIRMED` and `INVOICED` are successful states;
the portal may show `INVOICED` without an intermediate state being visible.
`FAILED`, `EXPIRED`, and `CANCELLED` do not authorize fulfillment.

Your integration must verify the trusted result and expected business terms
before fulfilling once. Use the [backend confirmation flow](goat-flow-developer-quickstart.md#5-read-the-payment-result-on-the-backend)
or the [verified webhook contract](#9-webhook-configuration).
Order and checkout/session status names are different; developers can find
the exact [order fields](goat-flow-api-reference.md#5-read-order-status) in the API Reference.

### 11.2 Order Reconciliation

**Payments → Reconciliation** (`/payments/reconciliation`) provides
accounting-oriented reports. Filter by
**Chain** and a **From/To** date range, then **Apply**. Summary cards show Total
Orders, Matched Payments (confirmed), Unpaid Orders (waiting or expired before
payment), and Late Payments (received after expiry).

![Order Reconciliation](./images/24-order-reconciliation.png)

Three report tabs — **Matched**, **Unpaid Orders**, and **Late Payments** — each list
rows with Order ID, Dapp Order, From/To Chain, Token, Amount, Status, Payment Tx,
Payout Tx, and Confirmed At. **Export CSV** exports the active tab under the current
filters. Amounts use the token's normal display units; no cross-token totals are shown.

`Payment Tx` and `Payout Tx` are portal column labels. For DIRECT transfers,
their presence does not imply that GOAT Flow holds or later disburses buyer funds.

---

## 12. QuickPay and Products

Use **Payment Setup → PayKit Settings** (`/acceptance/paykit-settings`) to
activate or pause PayKit. Use **QuickPay → Accept Any Amount**
(`/acceptance/paykit`) for a custom-amount public merchant link.
Crypto checkout requires an enabled DIRECT merchant, QuickPay enabled,
eligible receiving tokens, and the deployment's fee/limit configuration.
For Card, confirm fiat eligibility and provider configuration separately.

### 12.1 Configure QuickPay

1. Sign in as owner. In **PayKit Settings**, complete the displayed setup
   requirements, activate PayKit, and confirm its status is **Active**.
2. Verify each accepted chain/token, contract, decimals, receiving address, and limit.
3. For a custom-amount link, open **Accept Any Amount**. Set its display name,
   description, optional HTTPS logo, payment methods, and memo requirement.
4. Set the available usage limits for unpaid sessions, daily sessions, and
   maximum checkout amount.
5. Save, reload, confirm the link is **Live**, and open the copied public page while signed out.
   Verify the environment and displayed terms.

![Hosted Checkout Configuration](./images/55-new-merchant-quickpay.png)

Copy the payment page, `agent.md`, and `manifest.json` links from the portal.
Do not change hostnames to move a link between environments. If QuickPay is
unavailable, contact [Support@goat.network](mailto:Support@goat.network).

### 12.2 QuickPay Products

Products set a merchant-controlled decimal price. They can offer Crypto, Card,
or both when the merchant and product configuration support those rails.
Open **QuickPay → Sell Products** (`/acceptance/products`).

![QuickPay Products](./images/57-new-merchant-product.png)

1. Choose a stable product key (`A-Z`, `a-z`, digits, `.`, `_`, `:`, `~`, `-`;
   1–64 characters). Treat it as an integration identifier.
2. Enter the name, optional description/HTTPS image, positive decimal price,
   enabled state, and sort order.
3. Save and confirm the product is **Enabled** and **Live**.
4. Open the public page and verify its price and rail-specific terms. Test one
   checkout and reconcile its result before enabling fulfillment.

![Public QuickPay Checkout](./images/58-new-merchant-public-checkout.png)

Use custom amounts for buyer-controlled payments such as tips. Use a Product
or an authenticated Checkout Session for merchant-controlled prices.
For a Product button, developers can use the
[Checkout SDK](../goatx402-checkout/README.md#product-checkout).
The browser supplies the merchant and product key, not the price.

### 12.3 Public agent and CLI access

Share the portal's `agent.md` link with payer integrations. `manifest.json`
provides machine-readable payment methods, pricing, and routes. The buyer
does not need your merchant API credentials.

Agent installation, wallet/RPC configuration, payment commands, idempotency,
and recovery are maintained in the [PayKit README](../goatx402-quickpay/README.md).
Creating a Card link still requires a person to complete the hosted payment;
the link itself is not payment confirmation.

### 12.4 Paid API Routes (MPP)

This optional setup configures GOAT Flow's MPP profile for fixed-price API access.
It is not needed for ordinary hosted checkout. MPP is an independent open
protocol; GOAT Flow's profile uses its own JSON endpoints and signed receipts.

1. Define the protected method/path in your application. Convert it to the
   portal identifier, for example `GET /api/docs/protected` → `GET:api:docs:protected`.
2. Confirm Owner access, an enabled DIRECT merchant, MPP capability, and a
   receiving address for the intended supported chain/token.
3. Open **Developer → MPP Routes** (`/acceptance/mpp-routes`), select **Add Route**, and
   enter the route, pricing version, token/network, and fixed decimal amount.
4. Save; check the active row and public manifest for the exact route and terms.
5. Test payment and receipt verification on the protected resource. Reject
   missing, expired, wrong-merchant, wrong-route, and replayed receipts according
   to your intended policy.

![Active MPP Route](./images/60-new-merchant-mpp-route.png)

Use the active selector and API response for supported chains. Older Testnet3
helper text mentioning “Tempo only” is not an availability contract.
The [MPP guide](advanced/mpp.md) owns client, middleware, receipt, and recovery details.

<a id="13-balance-fees-and-topup"></a>

## 13. Balance, Fees, and Top-up

> **Environment note:** screenshots in this section show Testnet3. Mainnet fee
> amounts, supported chains and assets, limits, and top-up behavior may differ.
> Confirm the live Mainnet **GOAT Flow Fees** pages before launch, and do not perform a
> Mainnet top-up without authorization.

Open **GOAT Flow Fees → Fee Balance** to view service-fee credits; use
**GOAT Flow Fees → Top Up Fee Balance** to add service-fee credits.

![Testnet3 New Merchant Fee Balance](./images/48-new-merchant-fee-balance.png)

### 13.1 Fee Balance

**Fee Balance** is a prepaid service-fee credit ledger, separate from
buyer-to-merchant transfers. It is not a balance of customer funds.
The Testnet3 UI states that new order or session services can stop when the
credit balance is depleted. The active per-chain fee is displayed in **Platform
fee configuration**; do not assume one global price or reuse Testnet3 fees on
Mainnet.

### 13.2 Self-Service Top-up

The Testnet3 **Fee Top-up** page requires:

- a supported top-up destination chain
- an amount in USD
- a connected EVM wallet
- a supported USDT or USDC top-up asset

The page displays USDT and USDC as 1:1 USD for this Testnet3 top-up flow.
Supported chains, top-up assets, receiving wallets, limits, confirmation policy,
and crediting behavior can differ by environment.

![Testnet3 Fee Top-up](./images/66-new-merchant-topup.png)

Top-up history can be filtered by `PENDING`, `COMPLETED`, `EXPIRED`, or `FAILED`.
Confirm the final status and corresponding Fee Balance credit before relying on
a top-up. Do not infer refund or retry behavior from an empty form.

### 13.3 Balance Cards

| Card | Description |
| --- | --- |
| **Current fee balance** | Current prepaid service-fee credit balance |
| **Total charged** | Cumulative fees charged |
| **Total refunded** | Cumulative refunded fees |

### 13.4 Fee Configuration

The **Platform fee configuration** table is read-only: merchants can view
configured fee rows but cannot edit pricing.

Check the Mainnet table or API for active fees before launch. Testnet3 values are
examples only and must not be quoted as Mainnet pricing.

### 13.5 Transaction History

The transaction history table exposes amount, chain ID, order, description, and
date columns:

| Type | Description |
| --- | --- |
| **Fee charged** | Order or session fee deduction (e.g. order creation fee) |
| **Fee refunded** | A returned fee recorded by the active deployment |
| **Fee funds added** | Service-fee credit added to Fee Balance (e.g. a top-up) |

Do not assume that every cancellation or expiry refunds a fee, that every top-up
is reversible, or that fee events are posted at
a particular lifecycle state unless the active Mainnet policy documents that
behavior.

---

## 14. Audit Logs

Open **Audit Logs** to view operation records.

The page is read-only and shows events newest first. The current portal shows 30
rows per page; filter and export availability may vary.

Typical audit events include:

- Profile changes
- Receiving address additions and removals
- Merchant approval and operator capability changes
- API key rotation
- Webhook configuration changes
- QuickPay configuration and product changes
- MPP route changes
- Invite code creation and revocation
- Other deployment-defined account actions

Each record contains:

| Field | Description |
| --- | --- |
| Time | When the action occurred |
| Action | Operation type |
| Description | Human-readable summary |
| Details | Structured action details and changed values |
| IP | Source IP address |

Audit rows may contain addresses, account identifiers, and source IPs. Do not
share or publish unredacted audit data.

---

## Launch checks

- [ ] Merchant is approved and enabled; security, recovery, and team roles are reviewed.
- [ ] Production origins, credentials, products, receiving settings, fees, and
      webhooks were checked against the live deployment, separately from Testnet3.
- [ ] Crypto receiving chain/token/recipient configuration is correct; Card
      eligibility, currency, connected account, and provider test/live mode are confirmed.
- [ ] Merchant fee balance and top-up ownership cover expected traffic.
- [ ] API secrets are backend-only; any return URLs match the merchant allowlist.
- [ ] An environment-appropriate payment completed end to end. The backend checked
      expected terms and fulfilled once; the result can be reconciled.
- [ ] Failed, cancelled, expired, delayed, duplicate, and ambiguous attempts were
      exercised without duplicate payment or fulfillment. Webhooks, if used,
      passed the deployment-confirmed authentication and replay checks.
- [ ] Monitoring, escalation, and launch ownership are recorded.

Support: [Support@goat.network](mailto:Support@goat.network)
