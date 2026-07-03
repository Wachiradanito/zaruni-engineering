# ADR-0001: PostgreSQL over MySQL

**Status:** Accepted  
**Date:** Early 2025  
**Context:** Zaruni needed a relational database that could reliably handle financial transactions (escrow, wallet ledger, payment settlement) alongside flexible product and user data.

---

## Decision

We use PostgreSQL as our primary database.

## Reasoning

The decision came down to one non-negotiable requirement: **financial correctness**.

Zaruni holds real money. An escrow table that can return a phantom read during a concurrent `SELECT FOR UPDATE` sequence isn't acceptable when the consequence is a double-funded escrow or a wallet balance that doesn't match the ledger. PostgreSQL's transaction isolation and its row-level locking behaviour under `REPEATABLE READ` and `SERIALIZABLE` modes gave us guarantees MySQL's default `REPEATABLE READ` under InnoDB doesn't match in practice.

Three specific features sealed it:

**1. JSONB**  
Payment provider callbacks (M-PESA, Stripe) return payloads with shifting schemas. Storing them as `JSONB` lets us persist the raw payload without a migration every time Safaricom changes a field name, while still being able to index into it with GIN indexes when needed.

**2. `SELECT FOR UPDATE SKIP LOCKED`**  
Our webhook processing uses `select_for_update(skip_locked=True)` to ensure exactly one worker processes each callback. MySQL supports `SKIP LOCKED` from 8.0+, but Django's ORM support for it was more mature and predictable on Postgres at the time we built this.

**3. Partial indexes and expression indexes**  
We index things like `WHERE processed = false` (partial) to keep our webhook processing queries fast without indexing the entire table. MySQL's support for functional indexes is newer and less battle-tested in our experience.

## Alternatives Considered

**MySQL / MariaDB**  
Would have worked. The ecosystem is larger for shared hosting, and it's what many Django tutorials default to. The dealbreaker was operational: our team had more Postgres production experience, and the transaction semantics are better documented and more predictable for financial workloads.

**SQLite**  
Development only. Not viable for concurrent production workloads.

**MongoDB / NoSQL**  
Considered briefly for the flexible-schema appeal of payment payloads. Rejected because our core data (wallet, escrow, user accounts) is deeply relational — enforcing foreign keys and atomic multi-table transactions at the database level is worth more than schema flexibility.

## Consequences

**Better:**
- Strong transaction guarantees for all financial operations
- JSONB handles provider payload variety without schema migrations
- `SKIP LOCKED` works exactly as documented, no surprises
- Full-text search built-in (no Elasticsearch for basic search)

**Harder:**
- Slightly more complex to self-host than MySQL for developers coming from a shared-hosting background
- Fewer managed hosting options at the budget end of the market (though Hetzner managed Postgres and Supabase make this less of an issue now)

**Accepted tradeoff:**  
Operational complexity is worth the correctness guarantees at production scale.

---

*← Back to [decisions](../README.md#️-architecture-decision-records)*
