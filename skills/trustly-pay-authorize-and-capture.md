---
name: trustly-pay-authorize-and-capture
description: Accept a Pay by Bank payment with Trustly Pay in the US/Canada — establish a deferred bank authorization, wait for the Authorize webhook with the split token, then capture funds server-side and settle on the Completed event.
api: Trustly North America API
base_url: https://trustly.one/api/v1
sandbox_url: https://sandbox.trustly.one/api/v1
openapi: openapi/trustly-north-america-openapi.yml
operations:
  - transactions_post-establish
  - transactions_post-transactions-transactionId-capture
  - transactions_get-transaction
  - transactions_post-transactions-cancel
  - transactions_post-transactions-refund
webhooks:
  - eventNotifications_Establish
  - eventNotifications_Authorize
  - eventNotifications_Update
  - eventNotifications_Completed
  - eventNotifications_Failed
generated: '2026-09-18'
method: generated
source: https://amer.developers.trustly.com/integrate/accept-payments/trustly-pay
---

# Trustly Pay: authorize once, capture later

Trustly Pay is a two-phase flow. Phase 1 puts the user through the Lightbox (or Select Bank Widget) to
create a **bank authorization**; phase 2 is a server-to-server **capture** against that authorization.
Nothing moves until the capture, and captures settle over ACH, so the final state always arrives by webhook.

## Before you start
- Credentials: `accessId` / `accessKey` for the environment (sandbox and production are separate). Every call is HTTP Basic.
- Production requests and `establishData` need a `requestSignature` (Base64 HMAC-SHA1 over the ordered parameter list, keyed by `accessKey`). Generate it server-side; never put `accessKey` in the client. See `authentication/trustly-authentication.yml`.
- A public HTTPS `notificationUrl` that answers `200` within 3 seconds.

## Steps
1. **Establish** — `transactions_post-establish` (`POST /establish`), or let the client SDK call it via `Trustly.selectBankWidget(establishData, TrustlyOptions)`. Set `paymentType: "Deferred"` (this is the Trustly Pay value even for immediate payment; `Instant` is deprecated), a unique `merchantReference`, `currency`, `amount` (pass the real amount so funds are checked at authorization), `customer.externalId`, `returnUrl`, `cancelUrl`, `notificationUrl`. Keep PII out of `description`.
2. **User authorizes** in the Lightbox. Trustly redirects to `returnUrl` with a `transactionId` and POSTs an `Authorize` event (`eventType: Authorize`, `status: 2`). Validate the `Authorization` header before trusting it. Store the `splitToken` from that event — it is your half of the credential and is required on every capture.
3. **Capture** — `transactions_post-transactions-transactionId-capture` (`POST /transactions/{transactionId}/capture`) with `splitToken`, `amount` and a **unique** `merchantReference`. The immediate response is `Pending`; that is not settlement.
4. **Settle on webhook** — treat `Completed` (`eventNotifications_Completed`) as money received. `Failed`/`Deny` carry `paymentProviderTransaction.status` (SW021 insufficient funds, SW054/SW055 risk, SW057 expired split token — see `errors/trustly-decline-codes.yml`) and, when Available Funds Guidance is enabled, a `suggestedRetryAmount` you may re-try with.
5. **Read state on demand** with `transactions_get-transaction` (`GET /transactions/{transactionId}`) only for reconciliation; do not poll instead of listening.

## Idempotency and retries
- Replay protection is by `merchantReference`: a repeat on an Authorized/Processed/Completed transaction is ignored with internal code **210**. Reuse the same `merchantReference` when retrying a capture after a timeout; use a fresh one for a genuinely new capture.
- Refunds are **not** covered by this check — never blind-retry `transactions_post-transactions-refund`.

## Reversal
- Before the processing cut-off: `transactions_post-transactions-cancel`. If cancel errors, refund instead.
- After completion: `transactions_post-transactions-refund` (min 0.99, partial and repeated until the captured amount is exhausted; no calendar window is published).
- A user revoking bank access: cancel the authorization transaction; pending child captures stay and must be canceled individually.

## Sandbox
Demo Bank: any unique username + 3+ alphanumeric password succeeds; `NotEnoughFunds`, `ExpiredSplitToken`, `2FA`, `extendedreasoncode_10000` force the failure branches (`sandbox/trustly-sandbox.yml`).
