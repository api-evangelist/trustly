---
name: trustly-verify-account-and-identity
description: Verify a US/Canadian bank account and pull identity and balance data with Trustly — establish a Retrieval (data) authorization, wait for DataReady, then read the account balance, users, summary and Trustly ID details; fall back to micro-deposit verification or the account tokenize/verify endpoints.
api: Trustly North America API
base_url: https://trustly.one/api/v1
sandbox_url: https://sandbox.trustly.one/api/v1
openapi: openapi/trustly-north-america-openapi.yml
operations:
  - transactions_post-establish
  - accountData_get-transactions-account-balance
  - accountData_get-user
  - accountData_list-selected-accounts
  - accountData_get-transaction-account-summary
  - identity_get-trustly-id-user-data
  - identity_get-trustly-id-user-details
  - verifyCustomer_get-verify-customer
  - accounts_post-accounts-tokenize?-verifyAccount
  - accounts_get-accounts-verify
  - networkCheckApi_get-customer-lookup
webhooks:
  - eventNotifications_Authorize
  - eventNotifications_DataReady
  - eventNotifications_VerifyCustomer
generated: '2026-09-18'
method: generated
source: https://amer.developers.trustly.com/integrate/retrieve-data/verify-accounts-using-online-banking
---

# Account verification and identity data

Retrieval transactions fetch read-only data and move no money. The gate on every data endpoint is the
**DataReady** event: calling before it arrives is documented as returning partial data.

## Steps
1. **Establish a data authorization** — `transactions_post-establish` with `paymentType: "Retrieval"` (add the `verification` block per the establishData reference when you also want the identity check). The user logs in to their bank in the Lightbox.
2. **Wait for `Authorize`, then `DataReady`** (`eventNotifications_DataReady`). Only then call:
   - `accountData_get-transactions-account-balance` — `GET /transactions/{transactionId}/payment/paymentProvider/account/balance`
   - `accountData_list-selected-accounts` — `GET /transactions/{transactionId}/payment/paymentProvider/accounts`
   - `accountData_get-user` — `GET /transactions/{transactionId}/payment/paymentProvider/user`
   - `accountData_get-transaction-account-summary` — `GET /transactions/{transactionId}/accountSummary` (deposits, withdrawals and balances over time periods)
3. **Identity (Trustly ID)** — `identity_get-trustly-id-user-data` and `identity_get-trustly-id-user-details` (`GET /transactions/{transactionId}/user` and `/user/detail`); `verifyCustomer_get-verify-customer` returns the verification result, which also arrives as the `VerifyCustomer` event. Report outcomes back with `identity_post-transaction-feedback`.
4. **No online banking?** Use `accounts_post-accounts-tokenize?-verifyAccount` (`POST /accounts/tokenize`) and `accounts_get-accounts-verify` (`POST /accounts/verify`) with routing/account numbers, or the micro-deposit flow. Sandbox routing `124003116` with the documented account numbers drives the returned risk score and verification flag (`sandbox/trustly-sandbox.yml`).
5. **Is this user already known to Trustly?** `networkCheckApi_get-customer-lookup` (`GET /customers/lookup`) answers before you send them through the Lightbox.

## Notes
- All of this is read-only: there is nothing to reverse, and `merchantReference` idempotency does not apply to the data reads.
- Verification and capture should use different `merchantReference` values when you do both against one authorization.
