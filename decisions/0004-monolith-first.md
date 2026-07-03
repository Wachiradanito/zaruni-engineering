# ADR-0004: Modular Monolith over Microservices

**Status:** Accepted  
**Date:** Early 2025  
**Context:** Zaruni is a super-app with multiple domains: jobs, marketplace, services, delivery, payments, chat. The architecture question was whether to start with separate microservices (one per domain) or a single deployable unit (a monolith).

---

## Decision

We built a **modular monolith** — a single Django codebase with strict module boundaries enforced by import rules and architectural guidelines.

## Reasoning

Microservices are presented as "the right way to build at scale." The honest reality is that they solve problems most early-stage projects don't have yet, at the cost of complexity every project feels immediately.

**What microservices solve:**
- Independent deployment (update payments without touching jobs)
- Independent scaling (scale chat without scaling marketplace)
- Polyglot stacks (write chat in Go, payments in Python)
- Team autonomy (payments team owns their service end-to-end)

**What they cost:**
- Network calls between services (latency + failure modes)
- Distributed transactions (can't use database ACID across services)
- Service discovery and orchestration (K8s, service mesh, API gateways)
- Testing is harder (integration tests require multiple services running)
- Debugging is harder (trace spans across network boundaries)

At our current scale — one product, small team, under 100k requests/day — none of the benefits apply yet, and all of the costs are immediate.

The modular monolith gives us **extraction readiness** without paying the distributed systems tax upfront.

## How the Modular Monolith Works

The codebase is organised into Django apps by domain:

```
backend/
  accounts/
  wallet/
  payments/
  marketplace/
  jobs/
  services/
  deliver/
  chat/
  notifications/
```

Enforced rules:

**1. Cross-module imports must go through public service APIs**  
`marketplace.services` can call `wallet.services.fund_escrow`, but it cannot directly import models or utilities from wallet's internals. This keeps interfaces clean and makes extraction straightforward.

**2. Cross-module side effects use async events**  
When marketplace creates an order, it doesn't directly call `notifications.send_sms`. It emits an event: `emit_event('marketplace.order.created', payload)`. The notifications module subscribes to that event. This decouples the modules and works the same way whether they're in the same process or different services.

**3. Shared code lives in `core/`**  
Truly generic utilities (events, exceptions, verification registry, circuit breaker, fee policy) go in a `core/` package that all modules can import. This is the "shared kernel" in DDD terms.

These rules are documented in `docs/ARCHITECTURE.md` and enforced by code review. We've caught violations early when they're easy to fix.

The result: when we need to extract chat into its own service, the imports already go through a public API, the side effects already use events, and the domain logic is already isolated. Extraction becomes a deployment change, not a rewrite.

## Alternatives Considered

**Microservices from day one**  
Would have made early iteration slower. Moving fast on a super-app means changing the contracts between domains frequently — wallet needs a new escrow lifecycle event, jobs needs to query marketplace reputation, etc. Doing this across network boundaries while also building the features is cognitive overload.

**Pure monolith (no module boundaries)**  
Django's defaults encourage this — apps can import from each other freely. Rejected because it makes extraction impossible later without a major refactor. The discipline of module boundaries costs almost nothing upfront and compounds over time.

**Separate repos from day one**  
Considered for organisational cleanliness. Rejected because the domains share too much (authentication, wallet, event bus). Managing them as separate repos with shared dependencies is harder than managing them as apps in a monorepo.

## Consequences

**Better:**
- Fast iteration — changes that touch multiple domains deploy atomically
- Easy testing — all domains share a database in tests, no service mocks
- Simple deployment — one Docker image, one `docker-compose up`
- Cheap infrastructure — one server runs everything
- Clear contracts — module boundaries are enforced, extraction is straightforward later

**Harder:**
- Can't scale modules independently yet (all or nothing)
- Can't deploy modules independently (a payment bug requires redeploying everything, though Docker makes this fast)
- Module boundary discipline requires review enforcement (no automated guard rails in Django)

**When we'll revisit:**  
When a single domain (likely chat or payments) accounts for >50% of traffic and needs independent scaling, or when the team grows large enough that independent deployment would reduce coordination overhead. Not before.

---

*← Back to [decisions](../README.md#️-architecture-decision-records)*
