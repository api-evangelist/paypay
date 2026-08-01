---
name: Collect a payment with a PayPay dynamic QR code
description: Create a dynamic QR code, poll for the payment result, and refund if needed, using the PayPay Open Payment API (OPA v2).
api: openapi/paypay-opa-openapi-original.json
operations: [createQRCode, getCodePaymentDetails, refundPayment, getRefundDetails]
---

# Collect a payment with a PayPay dynamic QR code

Use the PayPay Open Payment API (OPA v2). Base gateway: `//apigw.paypay.ne.jp`
(sandbox `//apigw.sandbox.paypay.ne.jp`).

## Authentication
Every request is HMAC-SHA256 signed. Send
`Authorization: hmac OPA:{ApiKey}:{macData}:{nonce}:{epoch}:{hash}` and the merchant
identifier via the `X-ASSUME-MERCHANT` header (or `?assumeMerchant=`). The `epoch`
must be within 2 minutes of server time. See `authentication/paypay-authentication.yml`.

## Steps
1. **createQRCode** — POST `/v2/codes`. Provide a unique `merchantPaymentId`
   (<=64 chars), an `amount` object `{ amount, currency: "JPY" }`, and
   `codeType: "ORDER_QR"`. The response returns `codeId`, `url`, and `deeplink` —
   present these to the buyer. `merchantPaymentId` is the idempotency key: re-sending
   it will not double-charge (`DUPLICATE_DYNAMIC_QR_REQUEST` on conflict).
2. **getCodePaymentDetails** — GET `/v2/codes/payments/{merchantPaymentId}`. Poll
   every 4-5 seconds. Status moves `CREATED → COMPLETED` (instant) or `AUTHORIZED`
   (pre-auth). Prefer the transaction webhook over tight polling when available.
3. **refundPayment** (optional) — POST `/v2/refunds` with a unique `merchantRefundId`
   and the `paymentId`. Idempotent on `merchantRefundId`.
4. **getRefundDetails** — GET `/v2/refunds/{merchantRefundId}` to confirm the refund.

## Error handling
Read `resultInfo.code`. Handle `NO_SUFFICIENT_FUND`, `*_LIMIT_EXCEEDED`,
`PAY_METHOD_INVALIDATED` as buyer-actionable declines (see
`errors/paypay-decline-codes.yml`). On `RATE_LIMIT` (429) back off. Capture the
`X-REQUEST-ID` response header for support.
