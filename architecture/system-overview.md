# System Overview

Zaruni is a multi-service super-app running as a **modular monolith** — a single Django application with 25+ domain modules, strict import boundaries, and an event bus for cross-module communication.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Clients                                 │
│   React Native App (iOS/Android)  ·  Admin Dashboard (React)   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS
                    ┌──────▼──────┐
                    │    Caddy    │  Reverse proxy + automatic TLS
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌────▼──────┐
    │   HTTP API  │  │  WebSocket  │  │  Static   │
    │  (Gunicorn) │  │  (ASGI /    │  │   Files   │
    │    WSGI     │  │  Channels)  │  │           │
    └──────┬──────┘  └──────┬──────┘  └───────────┘
           └───────────────┬┘
                    ┌──────▼──────┐
                    │   Django    │  25+ domain modules
                    │  Monolith   │  Single codebase
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
    ┌──────▼──────┐  ┌──────▼──────┐  ┌────▼──────┐
    │ PostgreSQL  │  │    Redis    │  │  Celery   │
    │ + PgBouncer │  │             │  │  Workers  │
    └─────────────┘  └─────────────┘  └───────────┘
```

---

## Domain Modules

The application is organised into Django apps by domain. Each app owns its own models, services, views, serializers, and tests.

### Core Infrastructure
- **`core/`** — Shared utilities: event bus, exceptions, circuit breaker, fee policy, verification registry. No domain logic lives here. All domain modules may import from `core`, but `core` never imports from domain modules.

### Accounts & Identity
- **`accounts/`** — User registration, phone OTP, JWT auth, device verification, identity tiers, reputation, badges
- **`id_verification/`** — Document and identity verification workflows
- **`risk/`** — Risk scoring engine for users and transactions

### Financial
- **`wallet/`** — Wallet balances, escrow state machine, ledger, withdrawal limits
- **`payments/`** — Payment intents, M-PESA integration, Stripe, provider webhook ingestion, disputes
- **`finance/`** — Financial operations, platform fee configuration

### Marketplace & Commerce
- **`marketplace/`** — Shops, products, orders, reviews, discovery
- **`businesses/`** — Business entity management, operator accounts
- **`prices/`** — Price tracking and transparency

### Labour & Services
- **`jobs/`** — Gig job postings, applications, assignments, lifecycle
- **`services/`** — Service provider profiles, bookings, packages, scheduling

### Logistics
- **`deliver/`** — Delivery orchestration, rider assignment, tracking, SLA
- **`transit/`** — Transit/transport coordination
- **`maps/`** — Location services, geocoding, distance calculation
- **`location/`** — Location data (counties, sub-counties)

### Communication
- **`chat/`** — Real-time messaging (Django Channels + WebSockets)
- **`notifications/`** — Push, SMS, email, in-app notifications
- **`communications/`** — Communication templates and dispatch

### Trust & Safety
- **`proof/`** — QR/OTP proof-of-completion system
- **`moderation/`** — Content moderation
- **`support/`** — Customer support tools
- **`audit/`** — Audit logging
- **`receipts/`** — Transaction receipts

### Content & Discovery
- **`moments/`** — Social content (posts, stories)
- **`community/`** — Community features
- **`public_content/`** — Public-facing content pages
- **`public_discovery/`** — Public search and discovery

### Staff Operations
- **`staff/`** — Admin operations, staff tools

---

## Module Dependency Rules

The module hierarchy controls which app can import from which:

```
                  ┌─────────────┐
                  │    core/    │  ← Every module can import from here
                  └──────┬──────┘
          ┌───────────────┼───────────────┐
   ┌──────▼──────┐  ┌─────▼──────┐  ┌─────▼──────┐
   │  accounts   │  │   wallet   │  │marketplace │ ...
   └──────┬──────┘  └─────┬──────┘  └─────┬──────┘
          │               │               │
          │         ┌─────▼──────┐        │
          │         │  finance/  │        │
          │         │  payments/ │        │
          └─────────┴────────────┴────────┘
                          │
                   ┌──────▼──────┐
                   │notifications│  ← Never imported directly; receives events
                   └─────────────┘
```

**The rules:**
1. `core` is never imported by domain modules at the bottom of the hierarchy
2. Cross-domain calls go through the target module's **public service API** (`wallet.services.fund_escrow`, not internal wallet utilities)
3. Cross-domain side effects use **`emit_event()`** — notifications are never imported directly
4. `accounts` does not import from `marketplace`, `deliver`, or `jobs` at module load time (lazy imports inside functions only)

---

## Communication Patterns

There are three ways modules talk to each other. The choice depends on the consistency requirement:

### 1. Direct Service Call
Use when the result is needed immediately and must be atomic with the caller.

```python
# marketplace/services.py — needs escrow funded in the same transaction
from wallet.services import fund_escrow

escrow = fund_escrow(escrow=escrow, idempotency_key=f'order_{order.id}')
```

### 2. Async Event Bus
Use when the side effect is in a different domain and can tolerate async delivery.

```python
# wallet/services/escrow.py — notifying other modules of release
from core.events import emit_event

emit_event('wallet.escrow.released', {
    'escrow_id': str(escrow.id),
    'amount': str(result.net_amount),
    'payee_id': str(escrow.payee_id),
})
# notifications, audit, analytics all subscribe — wallet doesn't know or care
```

### 3. Django Signals
Use for same-app lifecycle hooks (post_save, post_delete) that must run synchronously.

---

## Data Layer

**PostgreSQL** is the primary database via PgBouncer connection pooling.

Key patterns used throughout:
- `select_for_update()` on all balance-modifying operations — prevents double-spend
- `select_for_update(skip_locked=True)` on queue-like processing (webhook ingestion, task dispatch)
- `get_or_create` with unique constraints for idempotency
- Partial indexes on high-traffic filtered queries (e.g. `WHERE processed = false`)

**Redis** is used for:
- Django cache (session data, M-PESA access token, rate limit counters)
- Celery broker (task queue)
- Django Channels channel layer (WebSocket message routing)

---

## Async Processing

**Celery** handles all async and scheduled work:

| Category | Examples | Retry policy |
|----------|----------|--------------|
| Financial | Wallet reconciliation, escrow auto-release | No auto-retry — manual recovery on failure |
| Cleanup | Stale media, expired sessions | Idempotent, retry 3× |
| Notifications | Push receipt processing | Retry 2× |
| Alerts | Stuck escrow detection, dispatch failures | Fire-and-forget |

Celery Beat runs scheduled tasks: daily reconciliation, midnight limit resets, periodic health checks.

---

## Real-time

**Django Channels** handles WebSocket connections for:
- In-app chat (persistent connections per conversation)
- Live delivery tracking
- Real-time notifications

WebSocket traffic is routed through Caddy to the ASGI server (same Django process, different worker class).

---

## External Integrations

| Service | Purpose |
|---------|---------|
| Safaricom Daraja (M-PESA) | STK Push payments, B2C payouts |
| Stripe | Card payments |
| FCM (Firebase) | Mobile push notifications |
| Google OAuth | Social login |
| Africa's Talking / Twilio | SMS |

All external payment callbacks are processed through the same idempotent webhook pipeline — see the [M-PESA idempotency case study](../case-studies/2025-mpesa-idempotency.md) for how duplicate delivery is handled.

---

*← Back to [README](../README.md)*
