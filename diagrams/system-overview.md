# Zaruni System Overview Diagram

> Full system architecture — clients, gateway, services, and data layer.

```mermaid
graph TB
    subgraph Clients["Clients"]
        APP["📱 React Native App\niOS / Android"]
        ADMIN["🖥️ Admin Dashboard\nReact + TypeScript"]
        WEB["🌐 Marketing Site\nAstro"]
    end

    subgraph Gateway["Gateway Layer"]
        CADDY["Caddy\nReverse Proxy + Auto HTTPS"]
    end

    subgraph App["Application Layer"]
        WSGI["Gunicorn\nHTTP API (WSGI)"]
        ASGI["Uvicorn\nWebSockets (ASGI)"]
    end

    subgraph Domains["Domain Modules (Django)"]
        direction LR
        ACCOUNTS["accounts\nauth · identity · reputation"]
        WALLET["wallet\nescrow · ledger · limits"]
        PAYMENTS["payments\nM-PESA · Stripe · webhooks"]
        MARKETPLACE["marketplace\nshops · products · orders"]
        JOBS["jobs\npostings · assignments"]
        SERVICES["services\nbookings · providers"]
        DELIVER["deliver\nlogistics · tracking"]
        CHAT["chat\nreal-time messaging"]
        NOTIFY["notifications\npush · SMS · email"]
        CORE["core\nevents · exceptions · fee policy"]
    end

    subgraph Data["Data Layer"]
        PG["PostgreSQL\n+ PgBouncer"]
        REDIS["Redis\ncache · broker"]
        CELERY["Celery Workers\n+ Beat Scheduler"]
    end

    subgraph External["External Services"]
        MPESA["Safaricom\nM-PESA Daraja"]
        STRIPE["Stripe"]
        FCM["Firebase\nPush Notifications"]
        SMS["Africa's Talking\nSMS"]
    end

    APP --> CADDY
    ADMIN --> CADDY
    WEB --> CADDY

    CADDY -->|HTTP /api/*| WSGI
    CADDY -->|WS /ws/*| ASGI
    CADDY -->|Static files| CADDY

    WSGI --> Domains
    ASGI --> CHAT

    Domains --> PG
    Domains --> REDIS
    Domains --> CELERY

    PAYMENTS --> MPESA
    PAYMENTS --> STRIPE
    NOTIFY --> FCM
    NOTIFY --> SMS
```

---

## Module Dependency Rules

```mermaid
graph TD
    CORE["core/\nevents · exceptions · fee_policy\ncircuit_breaker · verification"]

    ACCOUNTS["accounts/"]
    WALLET["wallet/"]
    MARKETPLACE["marketplace/"]
    JOBS["jobs/"]
    DELIVER["deliver/"]
    SERVICES["services/"]

    FINANCE["finance/"]
    PAYMENTS["payments/"]

    NOTIFY["notifications/"]

    CORE --> ACCOUNTS
    CORE --> WALLET
    CORE --> MARKETPLACE
    CORE --> JOBS
    CORE --> DELIVER
    CORE --> SERVICES

    WALLET --> FINANCE
    WALLET --> PAYMENTS

    MARKETPLACE -->|"wallet.services (public API only)"| WALLET
    JOBS -->|"wallet.services (public API only)"| WALLET
    DELIVER -->|"wallet.services (public API only)"| WALLET

    MARKETPLACE -->|"emit_event()"| NOTIFY
    JOBS -->|"emit_event()"| NOTIFY
    WALLET -->|"emit_event()"| NOTIFY

    style CORE fill:#f0f4ff,stroke:#4a6cf7
    style NOTIFY fill:#f0fff4,stroke:#22c55e
```

**Rules:**
- `core` is a shared kernel — imported by everyone, imports from no one
- Cross-domain calls go through public service APIs, never internal utilities
- `notifications` is never imported directly — receives events only

---

*Source: [architecture/system-overview.md](system-overview.md)*
