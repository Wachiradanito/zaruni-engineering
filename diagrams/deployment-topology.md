# Deployment Topology Diagrams

> Production infrastructure, request routing, and data flow.

---

## Production Stack

```mermaid
graph TB
    subgraph Internet["Internet"]
        USER["Users\n(Mobile App + Web)"]
        SAFARICOM["Safaricom\nM-PESA Callbacks"]
    end

    subgraph VPS["Hetzner VPS (Production)"]

        subgraph Proxy["Reverse Proxy"]
            CADDY["Caddy\nAuto HTTPS · Let's Encrypt\nHTTP/2"]
        end

        subgraph AppLayer["Application Layer"]
            GUNICORN["Gunicorn\nWSGI · HTTP API\n4 workers"]
            UVICORN["Uvicorn\nASGI · WebSockets\n2 workers"]
        end

        subgraph BackgroundJobs["Background Processing"]
            CELERY_W["Celery Workers\n3 processes · 4 threads each"]
            CELERY_B["Celery Beat\nScheduler"]
        end

        subgraph DataLayer["Data Layer"]
            PGBOUNCER["PgBouncer\nConnection Pool\n25 Postgres connections"]
            POSTGRES["PostgreSQL 15\nPrimary database"]
            REDIS["Redis\nCache · Broker · Channels"]
        end

        subgraph Observability["Observability"]
            PROMETHEUS["Prometheus\nMetrics"]
            GRAFANA["Grafana\nDashboards"]
            ALERTMANAGER["Alertmanager\nSlack alerts"]
        end

    end

    USER -->|"HTTPS"| CADDY
    SAFARICOM -->|"Webhook callbacks"| CADDY

    CADDY -->|"/api/*"| GUNICORN
    CADDY -->|"/ws/*"| UVICORN
    CADDY -->|"/static/*\n/media/*"| CADDY

    GUNICORN --> PGBOUNCER
    GUNICORN --> REDIS
    UVICORN --> REDIS

    CELERY_W --> PGBOUNCER
    CELERY_W --> REDIS
    CELERY_B --> REDIS

    PGBOUNCER --> POSTGRES

    GUNICORN --> PROMETHEUS
    CELERY_W --> PROMETHEUS
    PROMETHEUS --> GRAFANA
    PROMETHEUS --> ALERTMANAGER
```

---

## Request Routing Detail

```mermaid
flowchart LR
    subgraph Incoming["Incoming Request"]
        HTTP["HTTPS Request"]
    end

    subgraph Caddy["Caddy (Router)"]
        TLS["TLS Termination"]
        ROUTE{Route Match}
    end

    HTTP --> TLS --> ROUTE

    ROUTE -->|"/api/*\n/auth/*\n/admin/*"| GUNICORN["Gunicorn\nDjango WSGI"]
    ROUTE -->|"/ws/*"| UVICORN["Uvicorn\nDjango Channels ASGI"]
    ROUTE -->|"/static/*"| STATIC["Filesystem\n(collectstatic output)"]
    ROUTE -->|"/media/*"| MEDIA["Filesystem\n(user uploads)"]
```

---

## Celery Task Categories

```mermaid
graph TD
    BEAT["Celery Beat\n(Scheduler)"]

    subgraph Financial["Financial (Critical — no auto-retry)"]
        RECONCILE["reconcile_wallet_balances\nDaily 02:00 UTC"]
        WITHDRAWAL_RESET["reset_daily_withdrawal_limits\nMidnight UTC"]
    end

    subgraph Operational["Operational (Retry 3×)"]
        ESCROW_RELEASE["process_scheduled_escrow_releases\nEvery 5 min"]
        CLEANUP["cleanup_stale_pending_media\nHourly"]
    end

    subgraph Monitoring["Monitoring (Fire and forget)"]
        DISPATCH_ALERT["alert_on_dispatch_failures\nEvery 5 min"]
        STUCK_ESCROW["alert_stuck_escrows\nEvery 30 min"]
        HEARTBEAT["celery_heartbeat\nEvery 15 sec"]
    end

    BEAT --> Financial
    BEAT --> Operational
    BEAT --> Monitoring
```

---

## Database Connection Flow

```mermaid
flowchart LR
    subgraph Apps["Application Processes"]
        G1["Gunicorn\nWorker 1"]
        G2["Gunicorn\nWorker 2"]
        G3["Gunicorn\nWorker 3"]
        G4["Gunicorn\nWorker 4"]
        C1["Celery\nWorker 1"]
        C2["Celery\nWorker 2"]
        C3["Celery\nWorker 3"]
    end

    subgraph Pool["PgBouncer\nConnection Pool"]
        POOL["1000 client connections → 25 Postgres connections\ntransaction-pooling mode"]
    end

    PG["PostgreSQL\nmax_connections = 100"]

    G1 & G2 & G3 & G4 & C1 & C2 & C3 --> POOL
    POOL --> PG
```

Without PgBouncer, 7 app processes × Django connection per request would quickly exhaust PostgreSQL's connection limit under load.

---

## Deployment Process

```mermaid
flowchart TD
    PUSH["git push to main"]
    CI["GitHub Actions\nRun tests + build Docker image"]
    REGISTRY["Container Registry\nPush image with commit SHA tag"]
    DEPLOY["SSH to production\ndocker-compose pull"]
    MIGRATE["Run migrations\n(if schema changed)"]
    RESTART["docker-compose up -d\nRolling restart"]
    VERIFY["Health check\nGET /health/ → 200"]

    PUSH --> CI
    CI -->|"Tests pass"| REGISTRY
    REGISTRY --> DEPLOY
    DEPLOY --> MIGRATE
    MIGRATE --> RESTART
    RESTART --> VERIFY

    CI -->|"Tests fail"| BLOCKED["❌ Deploy blocked"]
```

---

*Source: [architecture/deployment-topology.md](deployment-topology.md)*
