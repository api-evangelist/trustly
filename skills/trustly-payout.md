---
name: trustly-payout
description: Send money to a consumer's bank account in the US/Canada with Trustly — reuse an existing bank authorization or collect one, then call Deposit and track it to Completed; reclaim if it must be pulled back.
api: Trustly North America API
base_url: https://trustly.one/api/v1
sandbox_url: https://sandbox.trustly.one/api/v1
openapi: openapi/trustly-north-america-openapi.yml
operations:
  - transactions_post-establish
  - transactions_post-transactions-deposit
  - transactions_post-transactions-transactionId-reclaim
  - transactions_get-transaction
  - transactions_post-transactions-refresh
webhooks:
  - eventNotifications_Authorize
  - eventNotifications_Update
  - eventNotifications_Completed
  - eventNotifications_Failed
generated: '2026-09-18'
method: generated
source: https://amer.developers.trustly.com/integrate/send-money/send-payouts-using-online-banking
---

# Payouts (Deposit)

In Trustly's vocabulary a **Deposit** moves money *from the merchant to the consumer* (a payout); a
**Capture** is the opposite. Standard payouts ride ACH (1–3 business days); instant payouts ride RTP or
FedNow when the receiving bank supports them.

## Steps
1. **Get an authorization.** If the user already authorized through Trustly Pay you have a `transactionId` and a stored `splitToken`; reuse them. Otherwise run `transactions_post-establish` with `paymentType: "Deferred"` and have the user select and log in to their bank, or use the account-information flow for payouts without online banking (see the send-money docs).
2. **Deposit** — `transactions_post-transactions-deposit` (`POST /transactions/{transactionId}/deposit`) with `splitToken`, `amount` and a unique `merchantReference`. The response is `Pending`.
3. **Track by webhook.** `Completed` means funds settled; `Failed`/`Deny` carry the decline status. In sandbox, payout notifications are batched every 5 minutes, so expect delay.
4. **Split token gone stale?** If a deposit fails with SW057 / code 326 (expired split token), call `transactions_post-transactions-refresh` (`POST /transactions/{transactionId}/refresh`) to refresh the bank authorization; if that fails, the user must log in again.

## Idempotency
`merchantReference` uniqueness is enforced on Deposit (duplicate -> code 210), so a retry with the same reference is safe and a second payout needs a new one.

## Reversal
`transactions_post-transactions-transactionId-reclaim` (`POST /transactions/{transactionId}/reclaim`) pulls a completed deposit back, in full or in parts of at least 0.99, until the amount is exhausted. It only works after the deposit has Completed; no calendar window is published. Reclaim is not covered by the duplicate-reference check.

## Do not
- Do not send a payout to an unverified account when your program requires verification; run `verifyCustomer_get-verify-customer` or the account-verification flow first.
- Do not treat the synchronous `Pending` as success.
