# Designing the Escrow State Machine

**Published:** July 2026  
**Module:** Wallet  
**Difficulty:** High — financial correctness, state safety, concurrent access

---

## The Problem

Zaruni is a marketplace and jobs platform. When a buyer purchases a product or hires a worker, there's a trust gap:

- **The buyer's fear:** "What if I pay and the seller never ships, or the worker never shows up?"
- **The seller/worker's fear:** "What if I deliver and the buyer claims they never received it, and I don't get paid?"

This isn't hypothetical — it's the default assumption in peer-to-peer transactions where neither party has an established reputation. Without a solution, transactions don't happen, or they happen off-platform where we can't provide any safety.

The solution is **escrow** — the platform holds the buyer's money, releases it to the seller only when both parties confirm the transaction is complete, and refunds it to the buyer if something goes wrong.

The question is: how do you design an escrow system that is **provably correct under concurrent access** and **impossible to leave in an inconsistent state**?

---

## Why It Mattered

An escrow bug can create or destroy money:

- **Funds released twice** → seller gets paid double, platform loses money
- **Funds released to the wrong party** → seller gets nothing, buyer loses trust
- **Funds stuck permanently** → neither party gets their money back, support nightmare
- **Concurrent refund and release both succeed** → money duplicated from thin air

At production scale, these aren't edge cases — they're certainties. If your state machine allows invalid transitions, concurrency will find them.

---

## The Investigation

Early prototypes used a simple status field:

```python
class Escrow(models.Model):
    status = models.CharField(
        choices=['pending', 'funded', 'released', 'refunded'],
    )
```

And state changes were just field assignments:

```python
def release_escrow(escrow):
    if escrow.status != 'funded':
        raise ValueError("Cannot release unfunded escrow")
    
    # credit seller wallet
    escrow.payee.wallet.balance += escrow.amount
    escrow.payee.wallet.save()
    
    # mark released
    escrow.status = 'released'
    escrow.save()
```

This worked in tests. It broke in production.

The failure mode: two workers processing separate "release" requests for the same escrow (legitimate if the buyer and seller both confirm completion simultaneously). Both pass the `if escrow.status != 'funded'` check before either commits, both credit the wallet, both mark it released.

Result: balance updated twice for the same escrow.

The root cause: **check-then-act with no lock**. The database guarantees nothing about what happens between the `SELECT` (the if-check) and the `UPDATE` (the save). Another transaction can modify the row between those two operations.

---

## The Solution

We needed three guarantees:

1. **Only valid transitions allowed** — `pending → funded → released` is valid, `released → funded` is not
2. **Transitions are atomic** — no concurrent modification possible during a transition
3. **Terminal states are irreversible** — once `released` or `refunded`, that's final

The design has four layers:

### Layer 1: Explicit Transition Rules

The allowed state transitions are defined as code:

```python
_VALID_TRANSITIONS = {
    'pending': ['funded', 'expired'],
    'funded': ['released', 'refunded', 'disputed'],
    'disputed': ['released', 'refunded'],
    'released': [],  # terminal
    'refunded': [],  # terminal
    'expired': [],   # terminal
}
```

Every state change goes through a validation function:

```python
def _can_transition(from_status, to_status) -> bool:
    return to_status in _VALID_TRANSITIONS.get(from_status, [])
```

Attempting an invalid transition raises an exception before any database write happens.

### Layer 2: Idempotency Keys for Terminal Transitions

Both `release_escrow` and `refund_escrow` require an idempotency key. The key is stored on the escrow when the transition succeeds:

```python
def release_escrow(*, escrow, idempotency_key: str):
    if _has_transition_key(escrow, idempotency_key):
        return  # already released with this key — no-op
    
    # ... perform release ...
    
    _remember_transition_key(escrow, idempotency_key)
```

If `release_escrow` is called twice with the same key, the second call is a no-op. If called twice with *different* keys, the second call fails the `_can_transition` check because `released` allows no further transitions.

This means: idempotent calls succeed silently, conflicting calls fail loudly.

### Layer 3: Row-Level Lock on All State Changes

Every function that modifies escrow state wraps the escrow fetch in `select_for_update()`:

```python
@transaction.atomic
def release_escrow(*, escrow, idempotency_key: str):
    locked_escrow = (
        Escrow.objects
        .select_for_update()
        .get(id=escrow.id)
    )
    
    # now we hold an exclusive lock on this row
    # no other transaction can read or modify it until we commit
```

This prevents the check-then-act race. If two workers try to release the same escrow:

- Worker A acquires the lock, proceeds
- Worker B blocks on `select_for_update()` until A commits
- By the time B gets the lock, the status is `released`
- B's `_can_transition('released', 'released')` fails

Only one worker ever performs the transition.

### Layer 4: Unique Constraint on (context_type, context_id)

Every escrow is tied to a specific transaction context — a marketplace order, a job assignment, a service booking. That context is unique:

```python
class Escrow(models.Model):
    context_type = models.CharField(max_length=50)
    context_id = models.CharField(max_length=50)
    
    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['context_type', 'context_id'],
                name='wallet_escrow_context_unique'
            )
        ]
```

This prevents a bug where two escrows are accidentally created for the same order. The database enforces one-escrow-per-transaction.

---

## The Outcome

After deploying the state machine with these four layers:

- **Zero invalid state transitions** — the transition map is enforced, no `released → funded` bugs
- **Zero double-release bugs** — the row lock prevents concurrent modifications
- **Zero double-creation bugs** — the unique constraint prevents duplicate escrows for the same context

The escrow system has processed thousands of transactions with no financial correctness incidents since this design shipped.

The pattern — transition map + idempotency keys + row locks + unique constraints — is now the template for every state machine in Zaruni. We've applied it to job lifecycles, service bookings, and dispute resolution.

---

## Code You Can Read

The actual implementation is in the private codebase, but the pattern is generic. If you're building a similar system:

1. Define valid transitions as data, not scattered in if-statements
2. Require idempotency keys on terminal transitions
3. Always `select_for_update()` before modifying state
4. Use database constraints as the final guard against application bugs

No clever algorithms, no distributed consensus protocol — just careful use of the tools databases already provide.

---

*← Back to [case studies](../README.md#-case-studies)*
