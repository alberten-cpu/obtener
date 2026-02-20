# Obtener Stripe Connect Implementation Overview

This document outlines how to implement Stripe Connect for **seller payouts**, **driver payouts**, commissions, delivery-fee allocation, and refund/reversal safety.

## 1) Core Financial Rules (Server-Side Source of Truth)

Implement all money movement from a single backend payout logic layer:

- **Product commission**: 5% of product subtotal **before GST**.
- **Retailer payout**: `product_subtotal - (5% pre-GST commission)`.
- **Driver payout**: fixed **$5** per completed order.
- **Delivery fee**:
  - Customer pays **$7**.
  - **$5** to driver.
  - **$2** retained by Obtener.
- **Stripe processing fees**:
  - Absorbed fully by Obtener.
  - Never deducted from seller or driver transfer calculations.

### Suggested Data Model Fields per Order

Store immutable money snapshots at payment time for auditability:

- `product_subtotal_ex_gst`
- `gst_amount`
- `delivery_fee_total` (= 700 cents)
- `platform_commission_amount` (5% pre-GST)
- `retailer_payout_amount`
- `driver_payout_amount` (= 500 cents)
- `platform_delivery_margin` (= 200 cents)
- `currency`
- `payment_intent_id`
- `seller_transfer_id` (nullable)
- `driver_transfer_id` (nullable)
- `refund_id` / `reversal_ids` (nullable)
- `financial_status` (e.g. `paid`, `rejected_refunded`, `completed_paid_out`)

## 2) Stripe Account Setup

### Stripe Connect Mode

- Enable **Stripe Connect (Express)** on platform account.
- Use destination account IDs for both:
  - sellers (retailers)
  - drivers

### Recommended Stripe Objects

- **Customer payment**: `PaymentIntent`.
- **Payouts to connected accounts**: `Transfer` objects (created by your backend after business conditions are met).
- **Refund on rejection**: `Refund` API against the payment.
- **Edge-case rollback**: `Transfer Reversal` if a transfer already happened.

## 3) Seller Onboarding Flow (Dashboard Trigger)

1. Seller clicks **Connect Stripe** in Seller Dashboard.
2. Backend creates or reuses Stripe `Account` (`type=express`).
3. Backend generates an onboarding `account_link`.
4. Frontend redirects seller to Stripe-hosted onboarding.
5. On return + webhook (`account.updated`), backend checks:
   - `charges_enabled`
   - `payouts_enabled`
   - requirements status
6. Mark seller as **payout-ready** in DB.

### API Endpoints (example)

- `POST /api/seller/stripe/connect`
- `GET /api/seller/stripe/status`
- `POST /webhooks/stripe`

## 4) Driver Onboarding Flow (App Trigger)

1. Driver signs up in mobile app.
2. Driver taps **Set Up Payout Account**.
3. Backend creates/reuses Express account for driver.
4. Backend returns onboarding link; app opens in webview/external browser.
5. Webhook and status check mark driver as **payout-ready**.

### API Endpoints (example)

- `POST /api/driver/stripe/connect`
- `GET /api/driver/stripe/status`
- `POST /webhooks/stripe`

## 5) Payment, Commission, and Payout Execution

### At Checkout

- Create `PaymentIntent` for full charge amount (products + GST + delivery).
- Persist an order-level financial snapshot before capture/finalize.

### On Successful Payment

- Set order to `paid`.
- Do **not** transfer immediately unless order conditions are satisfied.

### On Order Completion

Run payout engine transactionally:

1. Validate order eligible for payout and not refunded.
2. Create seller transfer for `retailer_payout_amount`.
3. Create driver transfer for fixed $5.
4. Save transfer IDs and move order to payout-complete state.
5. Ensure idempotency keys per payout action.

## 6) Rejection, Refund, and Reversal

### Rejected Before Payout

- Trigger full refund (`Refund` for total customer payment).
- Mark order `rejected_refunded`.
- Skip all transfer creation.

### Rejected After Payout (Edge Case)

- Trigger full customer refund.
- Reverse seller transfer (full or calculated required amount).
- Reverse driver transfer if already sent.
- Record reversal IDs and reconciliation notes.

## 7) Webhook & Idempotency Requirements

Critical events to process:

- `payment_intent.succeeded`
- `charge.refunded` / `refund.updated`
- `account.updated` (onboarding readiness)

Safety controls:

- Verify Stripe signatures.
- Idempotency table keyed by Stripe event ID.
- Exactly-once payout guard (order-level lock/transaction).
- Dead-letter/retry strategy for transient failures.

## 8) Reconciliation & Audit Controls

Implement a ledger-style financial table (append-only preferred):

- entry type: charge / commission / transfer / refund / reversal
- amount, currency, Stripe object IDs
- order ID + actor ID (seller/driver)
- timestamps + status

Build daily reconciliation job:

- Compare internal ledger totals vs Stripe balance/transfer/refund data.
- Flag mismatches for manual review.

## 9) Suggested Implementation Phases

1. **Connect enablement + webhook skeleton**
2. **Seller onboarding endpoints + dashboard wiring**
3. **Driver onboarding endpoints + app wiring**
4. **Payout engine with commission/delivery split rules**
5. **Rejection refund + transfer reversal logic**
6. **Reconciliation jobs + operational dashboards**
7. **End-to-end Stripe test mode validation**

## 10) Testing Checklist

- Seller onboarding success/failure/requirements due.
- Driver onboarding success/failure/requirements due.
- Successful order:
  - retailer gets full computed payout (no Stripe fee deduction)
  - driver gets $5
  - platform retains 5% pre-GST + $2 delivery margin
- Rejected pre-payout => full refund + no transfers.
- Rejected post-payout => full refund + transfer reversals.
- Webhook replay/idempotency.
- Concurrency test: duplicate completion events do not double-pay.

## 11) Delivery Plan (Estimate)

- Connect setup/configuration: **3–4 days**
- Seller onboarding: **2–3 days**
- Driver onboarding: **2–3 days**
- Commission/payout engine: **4–5 days**
- Refund/reversal handling: **3–4 days**
- Stripe test mode + QA: **3–4 days**

Expected timeline: **~4–5 weeks** including validation hardening.

## 12) Key Risks / Dependencies

- Connect capability and country KYC requirements.
- Reliable webhook delivery + retry strategy.
- Strict idempotency to prevent duplicate payouts.
- Reconciliation confidence before production rollout.
