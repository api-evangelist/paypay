---
name: Pre-authorize and capture a PayPay payment
description: Authorize a PayPay payment now and capture (or update/void) it later using the OPA v2 pre-auth-capture flow.
api: openapi/paypay-opa-openapi-original.json
operations: [createQRCode, capturePaymentAuth, reauthorizePayment, revertAuth]
---

# Pre-authorize and capture a PayPay payment

Use the PayPay Open Payment API (OPA v2) pre-auth-and-capture flow when you charge
after fulfillment. Requires a merchant contract that supports Pre-Auth-Capture.

## Authentication
HMAC-SHA256 signed requests with `Authorization: hmac OPA:...` and
`X-ASSUME-MERCHANT`. See `authentication/paypay-authentication.yml`.

## Steps
1. **createQRCode** — POST `/v2/codes` with `isAuthorization: true` and an
   `authorizationExpiry`. On buyer approval the payment becomes `AUTHORIZED`
   (a `Transaction`/`AUTHORIZED` webhook fires). If the merchant is not enabled you
   get `PRE_AUTH_CAPTURE_UNSUPPORTED_MERCHANT`; an expiry beyond the max returns
   `PRE_AUTH_CAPTURE_INVALID_EXPIRY_DATE`.
2. **reauthorizePayment** (optional) — POST `/v2/payments/reauthorize` to change the
   authorized amount. Max 10 attempts (`REAUTH_MAX_LIMIT_EXCEEDED`); a higher amount
   may return `201 USER_CONFIRMATION_REQUIRED` requiring buyer confirmation.
3. **capturePaymentAuth** — POST `/v2/payments/capture` with a unique
   `merchantCaptureId` to move `AUTHORIZED → COMPLETED`. Watch for `ALREADY_CAPTURED`,
   `ORDER_EXPIRED`, `LIMIT_EXCEEDED`, and `NO_SUFFICIENT_FUND`.
4. **revertAuth** — POST `/v2/payments/preauthorize/revert` with a unique
   `merchantRevertId` to void an authorization (`AUTHORIZED → CANCELED`).

## Notes
Every write operation is made idempotent by its merchant-supplied id
(`merchantPaymentId`, `merchantCaptureId`, `merchantRevertId`). Reconcile via the
daily transaction CSV webhook. See `conventions/paypay-conventions.yml`.
