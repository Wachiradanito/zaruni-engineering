# Solving M-PESA Callback Race Conditions

**Published:** July 2026  
**Module:** Payments  
**Difficulty:** High — financial correctness, concurrency, external system behaviour

---

## The Problem

Zaruni handles real M-PESA payments. When a user pays via STK Push (the "enter your M-PESA PIN" prompt on your phone), Safaricom sends an asynchronous callback to our server once the payment is confirmed or rejected.

The problem: **Safaricom can and does send the same callback more than once.**

This is not a bug in their system — it's intentional. Their documentation explicitly states callbacks may be retried on network failures or timeouts. From their perspective, delivering the callback twice is safer than not delivering it at all. From our perspective, processing it twice meant:

- A balance updated twice for the same payment
- An escrow funded twice from the same transaction
- A user receiving double the value they paid for

None of those outcomes are acceptable on a platform handling real money.

---

## Why It Mattered

The consequence of processing a duplicate callback without protection:

1. User pays KES 500 via M-PESA
2. Callback arrives, balance updated KES 500 ✅
3. Network glitch on Safaricom's side — they retry the callback
4. Same callback arrives again, balance updated another KES 500 ❌
5. User now has KES 1,000 but only paid KES 500

At low transaction volume this might happen rarely. At production scale, with thousands of payments per day, it becomes a certainty — not a possibility. A platform that can silently create money from failed network calls is not a platform anyone can trust.

---

## The Investigation

The issue surfaced during load testing. We simulated M-PESA callback delivery under poor network conditions and noticed wallet balances didn't always match expected payment totals. Specifically, some test wallets ended up with double the credited amount.

Digging in, the flow at the time looked like this:

```
Safaricom → POST /payments/webhook/mpesa/
    → Parse callback payload
    → Find matching payment intent
    → Mark intent as confirmed
    → Credit wallet
```

The problem was in that last step. There was no check on whether the intent had *already* been confirmed. If two callbacks arrived close together — say within milliseconds of each other, which is realistic under retry conditions — both could pass through the "find matching intent" step before either had committed the state change to the database.

A classic **check-then-act** race condition.

---

## The Solution

We needed a design that made duplicate processing **structurally impossible**, not just unlikely.

The solution has three layers working together:

### Layer 1: Unique Constraint on the Webhook Log

Every incoming callback is logged before any processing begins. The log table has a `UNIQUE` constraint on `(provider, event_id)`:

```python
class ProviderWebhookLog(models.Model):
    provider    = models.CharField(max_length=30)
    event_id    = models.CharField(max_length=120)
    processed   = models.BooleanField(default=False)
    processed_at = models.DateTimeField(null=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['provider', 'event_id'],
                name='payments_provider_event_unique'
            )
        ]
```

The first callback to arrive creates the log row. The second callback — whether it arrives one millisecond or one hour later — hits the unique constraint on insert. We use `get_or_create` to handle this gracefully:

```python
log, created = ProviderWebhookLog.objects.get_or_create(
    provider=provider,
    event_id=event_id,
    defaults={'payload': payload},
)

if log.processed:
    return log  # already done — exit early
```

If the log already exists **and** is already processed, we return immediately. No double processing.

### Layer 2: Row-Level Lock Before Processing

The unique constraint handles the "already processed" case. But what about two callbacks arriving at the exact same millisecond, both finding the log row unprocessed?

For this, we use `SELECT FOR UPDATE SKIP LOCKED`:

```python
locked_log = (
    ProviderWebhookLog.objects
    .select_for_update(skip_locked=True)
    .filter(id=log.id, processed=False)
    .first()
)

if locked_log is None:
    # Another worker has the lock — they'll handle it
    return log
```

`select_for_update(skip_locked=True)` means: "lock this row exclusively, but if it's already locked by someone else, return nothing instead of waiting."

If two workers race to process the same callback:
- Worker A acquires the lock — proceeds with processing
- Worker B calls `select_for_update(skip_locked=True)` — gets `None` — exits

Only one worker ever processes a given callback. The database enforces this, not application logic.

### Layer 3: Idempotency Keys on All Financial Operations

Even with the webhook deduplication, downstream operations (balance updates, escrow state changes) have their own idempotency keys. Every financial mutation writes an `idempotency_key` field — a unique string scoped to the specific operation:

```python
# Scoped to this specific intent's funding operation
idempotency_key = f'payment_intent_fund_{intent.id}'

existing = LedgerEntry.objects.filter(
    idempotency_key=idempotency_key
).first()

if existing:
    return existing  # this exact operation already ran
```

This is belt-and-suspenders: even if a bug somehow bypassed the webhook deduplication, the financial operations themselves would refuse to run twice for the same key.

---

## The Outcome

After deploying this design, duplicate callback processing dropped to zero — not reduced, eliminated.

The three-layer approach — unique constraint, row lock, idempotency keys — means the system stays correct under any delivery pattern Safaricom uses:

- Same callback delivered twice within milliseconds ✅ handled by row lock
- Same callback delivered an hour later ✅ handled by unique constraint + processed flag
- Subtle bug bypasses webhook layer ✅ handled by per-operation idempotency keys

The underlying principle is that **correctness shouldn't depend on external systems behaving well**. Safaricom retrying callbacks isn't a bug to report — it's expected behaviour to design around.

This pattern (log → lock → idempotency key) is now the standard for every external payment callback in Zaruni, including Stripe and M-PESA withdrawals.

---

## What We Extracted

The core callback-logging and idempotency pattern is generic enough to work in any Django payment integration — no Zaruni-specific logic.

It's part of **django-mpesa**, our open-source M-PESA integration package.

*More extracted tools: [extracted-tools/README.md](../extracted-tools/README.md)*

---

*← Back to [case studies](../README.md#-case-studies)*
