# Payments Architecture

Zaruni handles real money — M-PESA payments from Kenyan users, escrow holds for marketplace and job transactions, and B2C payouts to workers and sellers. This document describes the payment system design.

---

## Design Principles

1. **No direct transfers** — all money movement goes through the escrow state machine, never a direct wallet-to-wallet transfer
2. **Idempotency everywhere** — every financial operation has an idempotency key; running it twice is a no-op
3. **Escrow first, release after** — funds are held until both parties confirm the transaction is complete
4. **External callbacks are untrusted** — M-PESA callbacks are logged, locked, and deduplicated before any processing

---

## Components

```
┌────────────────────────────────────────────────────────────────┐
│                    payments/                                   │
│  PaymentIntent  ·  ProviderWebhookLog  ·  EscrowDispute       │
│  M-PESA provider  ·  Stripe provider  ·  intent services      │
└────────────────────────┬───────────────────────────────────────┘
                         │ funds / releases
┌────────────────────────▼───────────────────────────────────────┐
│                    wallet/                                     │
│  Wallet  ·  Escrow (state machine)  ·  LedgerEntry            │
│  fund_escrow  ·  release_escrow  ·  refund_escrow             │
│  withdraw_from_wallet  ·  deposit_to_wallet                   │
└────────────────────────────────────────────────────────────────┘
```

---

## STK Push Payment Flow (M-PESA)

The most common payment path: a user pays for a marketplace order, job, or service via M-PESA.

```
User                  Zaruni API              Safaricom
 │                        │                       │
 │  POST /checkout        │                       │
 │───────────────────────►│                       │
 │                        │  STK Push Request     │
 │                        │──────────────────────►│
 │                        │  CheckoutRequestID    │
 │                        │◄──────────────────────│
 │  "Check your phone"    │                       │
 │◄───────────────────────│                       │
 │                        │                       │
 │  [User enters PIN]     │                       │
 │                        │  Callback POST        │
 │                        │◄──────────────────────│
 │                        │                       │
 │                        │  [idempotent process] │
 │                        │  wallet credited      │
 │                        │  escrow funded        │
 │  Notification: paid ✓  │                       │
 │◄───────────────────────│                       │
```

### What happens in "idempotent process"

This is the critical section — see the [M-PESA idempotency case study](../case-studies/2025-mpesa-idempotency.md) for the full story. In brief:

1. Callback arrives at `/payments/webhook/mpesa/`
2. IP allowlist check — non-Safaricom IPs are rejected
3. `ProviderWebhookLog.objects.get_or_create(provider, event_id)` — unique constraint prevents duplicate rows
4. `select_for_update(skip_locked=True)` — only one worker can process a given callback
5. Provider adapter parses the payload, extracts confirmation status
6. Intent matched via `CheckoutRequestID`
7. Wallet credited with idempotency key scoped to this intent
8. Escrow funded (if payment was for an escrow-backed transaction)
9. Log marked `processed = True`

If Safaricom delivers the callback twice, step 3 returns the existing row and step 4 returns `None` for the second worker. No double credit possible.

---

## Escrow State Machine

All marketplace, job, and service transactions use escrow — funds held by the platform, released only when both parties confirm completion.

```
  create_escrow()
        │
        ▼
  ┌──────────┐
  │ pending  │
  └────┬─────┘
       │ fund_escrow()
       ▼
  ┌──────────┐
  │  funded  │──────────────────────────────────┐
  └────┬─────┘                                  │
       │                              refund_escrow()
       │ release_escrow()                        │
       ▼                                         ▼
  ┌──────────┐                           ┌──────────┐
  │ released │                           │ refunded │
  └──────────┘                           └──────────┘
  (terminal)                             (terminal)

  + expired (from pending, via scheduled task)
```

**Invariants:**
- `fund_escrow` is idempotent — calling twice on a funded escrow is a no-op
- `release_escrow` and `refund_escrow` are both idempotent — guarded by transition key
- `select_for_update()` wraps all state transitions — no concurrent modifications
- `payer != payee` is enforced on creation — no self-escrow

**Who creates escrows:**
- `marketplace.services` — product orders
- `jobs.services` — job assignments
- `services.services` (service bookings)
- `deliver.services` — delivery jobs

All of them call `wallet.services.create_escrow` and `wallet.services.fund_escrow` through the public wallet API. None of them touch the Escrow model directly.

---

## Wallet Ledger

Every credit and debit is a `LedgerEntry` row — an append-only log of all balance movements. The wallet's `balance` is always derivable from the sum of its ledger entries.

A periodic reconciliation task (Celery Beat, runs nightly) verifies that computed balances match stored balances. Any discrepancy triggers an alert.

All balance-modifying operations use `select_for_update()` on the wallet row to prevent concurrent modifications producing incorrect balances.

---

## Withdrawal Flow

When a seller or worker withdraws their wallet balance:

1. User requests withdrawal (specifying amount and M-PESA phone number)
2. Withdrawal limit checked — daily limit enforced, resets at midnight
3. Balance deducted atomically with `select_for_update()`
4. B2C M-PESA payout initiated with idempotency key
5. Payout status polled / callback received
6. On failure: balance refunded, user notified

Withdrawal limits exist to protect against account compromise and fraud. New accounts have lower limits that increase as trust score improves.

---

## Dispute System

When a transaction goes wrong, either party can raise a dispute:

- Buyer: non-delivery, damaged item, service not rendered
- Seller: unfair refund request, buyer no-show

Disputes put the associated escrow on hold. A staff member reviews evidence submitted by both parties and resolves in favour of one side. Resolution either releases the escrow to the seller or refunds it to the buyer.

---

## Provider Architecture

Zaruni supports multiple payment providers through a provider adapter pattern:

```python
class PaymentIntentProviderAdapter:
    def initiate(self, *, intent, payload) -> PaymentInitiationResult: ...
    def verify_webhook(self, *, payload, headers) -> bool: ...
    def process_webhook(self, *, payload) -> PaymentWebhookResult: ...
    def get_status(self, *, intent) -> PaymentStatusResult: ...
    def refund(self, *, intent, amount) -> dict: ...
```

Current providers: M-PESA (Daraja), Stripe. Adding a new provider means implementing this interface — no changes to the core payment flow.

---

*← Back to [README](../README.md)*
