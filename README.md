# Obtener — Stripe Connect Implementation Starter

This repository now contains a **practical starter plan** to implement Stripe Connect for both sellers and drivers, aligned to your confirmed payout model.

## What was done in this update

- Converted the previous high-level overview into a **“start now” execution guide**.
- Added a **Week 1 implementation order** so engineering can begin immediately.
- Added **concrete backend contracts** (DB fields + API endpoints + webhook handling sequence).
- Added payout/refund rules with exact formulas and idempotency protections.
- Added a clear **definition of done** and test checklist for go-live readiness.

## Financial Rules (Locked)

All calculations must be done server-side from a single payout service.

- Product commission = **5% of pre-GST product subtotal**.
- Retailer payout = `product_subtotal_ex_gst - commission`.
- Driver payout = **$5.00** per completed order.
- Delivery fee charged to customer = **$7.00**:
  - Driver receives **$5.00**
  - Obtener keeps **$2.00**
- Stripe processing fees are absorbed by Obtener (never deducted from retailer/driver payouts).

## How to Start (Implementation Order)

### Step 1 — Stripe Platform Setup (Day 1)

1. Enable Stripe Connect (Express) in platform account.
2. Create environment variables:
   - `STRIPE_SECRET_KEY`
   - `STRIPE_WEBHOOK_SECRET`
   - `STRIPE_CONNECT_REFRESH_URL`
   - `STRIPE_CONNECT_RETURN_URL`
3. Add webhook endpoint in Stripe dashboard for:
   - **Endpoint URL (production):** `https://<your-backend-domain>/stripe/webhook`
   - Example: `https://api.obtener.com/stripe/webhook`
   - If your backend uses an `/api` prefix, use: `https://<your-backend-domain>/api/stripe/webhook`
   - For local Stripe CLI testing: `http://localhost:<port>/stripe/webhook`
   - `payment_intent.succeeded`
   - `charge.refunded`
   - `refund.updated`
   - `account.updated`

> Important: Use your **backend server URL** (not frontend URL) because Stripe must send events directly to your server webhook handler.

### Step 2 — Database Financial Snapshot (Day 1–2)

At order payment time, store immutable values:

- `product_subtotal_ex_gst`
- `gst_amount`
- `delivery_fee_total`
- `platform_commission_amount`
- `retailer_payout_amount`
- `driver_payout_amount`
- `platform_delivery_margin`
- `currency`
- `payment_intent_id`
- `seller_transfer_id` (nullable)
- `driver_transfer_id` (nullable)
- `refund_id` (nullable)
- `seller_reversal_id` (nullable)
- `driver_reversal_id` (nullable)
- `financial_status`

### Step 3 — Seller Onboarding API (Day 2)

Implement:

- `POST /api/seller/stripe/connect`
  - create or reuse Stripe Express account
  - generate account onboarding link
  - return URL to frontend
- `GET /api/seller/stripe/status`
  - return `charges_enabled`, `payouts_enabled`, pending requirements

Dashboard action:

- **Connect Stripe** button calls `/api/seller/stripe/connect` and redirects user.

### Step 4 — Driver Onboarding API (Day 2–3)

Implement:

- `POST /api/driver/stripe/connect`
- `GET /api/driver/stripe/status`

Driver app action:

- **Set Up Payout Account** button opens onboarding URL in webview/browser.

### Step 5 — Payout Engine (Day 3–5)

On order completion only:

1. Check order is paid, not refunded, not already transferred.
2. Create seller transfer for `retailer_payout_amount`.
3. Create driver transfer for `$5`.
4. Save transfer IDs in DB.
5. Mark `financial_status=completed_paid_out`.

Idempotency requirements:

- Use Stripe idempotency keys for every transfer/refund/reversal call.
- Store processed webhook `event_id` and skip reprocessing.

### Step 6 — Rejection/Refund Handling (Day 5)

- If order rejected before payout:
  - create full refund to customer
  - mark `financial_status=rejected_refunded`
  - do not create transfers
- If rejected after payout (edge case):
  - refund customer fully
  - reverse seller transfer
  - reverse driver transfer
  - store reversal IDs and reconciliation note

## Payout Calculation Formula (Reference)

Assume:

- product subtotal ex GST = `P`
- commission = `0.05 * P`
- retailer payout = `P - commission`
- driver payout = `5.00`
- platform delivery margin = `2.00`

Then platform revenue for an order (excluding Stripe fees) is:

- `commission + 2.00`

## Webhook Processing Sequence

1. Verify Stripe signature.
2. Check if event ID already processed.
3. Process event atomically per order.
4. Write ledger entry (charge/refund/transfer/reversal).
5. Mark event as processed.

## Definition of Done

Implementation is complete only when all are true:

- Seller onboarding works end-to-end and account becomes payout-enabled.
- Driver onboarding works end-to-end and account becomes payout-enabled.
- Completed order triggers exactly one seller transfer + one driver transfer.
- Rejected order triggers full refund and no payout leakage.
- Edge-case reversal works when payout happened before rejection.
- Webhook replay does not duplicate financial movement.
- Daily reconciliation report balances against Stripe data.

## Test Checklist

- Seller onboarding happy path + incomplete KYC path.
- Driver onboarding happy path + incomplete KYC path.
- Successful order payout math verification.
- Rejected pre-payout full refund.
- Rejected post-payout transfer reversal.
- Duplicate webhook replay safety.
- Concurrent completion requests do not double-pay.

## Delivery Estimate

- Connect setup/config: 3–4 days
- Seller onboarding: 2–3 days
- Driver onboarding: 2–3 days
- Payout engine: 4–5 days
- Refund/reversal: 3–4 days
- QA + Stripe test mode: 3–4 days

Total: ~4–5 weeks with validation.
