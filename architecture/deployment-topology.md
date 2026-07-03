# Deployment Topology

Zaruni runs on self-hosted VPS infrastructure (Hetzner). This document describes the production stack — what runs where, how services talk to each other, and the operational tooling.

---

## Stack Diagram

```
Internet
   │
   ▼
┌──────────────┐
│    Caddy     │  Reverse proxy + automatic HTTPS
└──────┬───────┘
       │
       ├──► /api/*     ──► Django (Gunicorn WSGI)
       ├──► /ws/*      ──► Django (Uvicorn ASGI for WebSockets)
       ├──► /static/*  ──► Filesystem (served by Caddy)
       └──► /admin/*   ──► Django admin (same Gunicorn process)
           │
    ┌──────┴────────────────────────┐
    │      Django Application        │
    │  25+ domain modules, one proc  │
    └──────┬──────────────┬──────────┘
           │              │
    ┌──────▼──────┐  ┌───▼───────┐
    │ PostgreSQL  │  │   Redis   │
    │ + PgBouncer │  └───┬───────┘
    └─────────────┘      │
                    ┌────▼────────┐
                    │   Celery    │  Async workers + Beat scheduler
                    │   Workers   │
                    └─────────────┘
```

---

## Services

### Caddy (Reverse Proxy)

**Purpose:** HTTPS termination, routing, static file serving  
**Runs:** Single process, systemd-managed  
**Config:** `/etc/caddy/Caddyfile`

Caddy automatically acquires and renews TLS certificates from Let's Encrypt. No manual `certbot` or cron jobs required. It routes:
- `/api/*` → Gunicorn (WSGI) for HTTP API requests
- `/ws/*` → ASGI server for Django Channels WebSockets
- `/static/*` → filesystem (static assets collected by `collectstatic`)
- `/media/*` → filesystem (user-uploaded files)

**Why Caddy:** See [ADR-0002: Caddy over Nginx](../decisions/0002-caddy-over-nginx.md)

---

### Django (Application Server)

**Purpose:** Core application logic  
**Runs:** Gunicorn (WSGI) + Uvicorn workers (ASGI) managed by systemd or Docker Compose  
**Workers:** 4 Gunicorn workers (WSGI), 2 Uvicorn workers (ASGI)

Gunicorn serves standard HTTP traffic. Uvicorn (via Django Channels) handles WebSocket connections for chat and real-time features. Both processes run the same Django codebase — the worker class differs.

**Container:** Single Docker image built from `backend/Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python manage.py collectstatic --noinput
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

### PostgreSQL + PgBouncer

**Purpose:** Primary database  
**Runs:** PostgreSQL 15 with PgBouncer in transaction-pooling mode

**Why PgBouncer:** Django opens a new connection on every request by default (in the common config). Under load, this exhausts Postgres's connection limit (`max_connections = 100` typically). PgBouncer sits in front, reusing a pool of Postgres connections for many client connections.

**Config:**
- `pool_mode = transaction` — connection returned to pool after each transaction
- `max_client_conn = 500` — clients can ask for up to 500 connections
- `default_pool_size = 25` — but PgBouncer maintains only 25 actual Postgres connections

**Backups:** Automated daily backups to Hetzner Object Storage via `pg_dump` cron job. Retention: 7 daily + 4 weekly.

---

### Redis

**Purpose:** Cache, Celery broker, Django Channels layer  
**Runs:** Single Redis instance (systemd-managed or Docker)

Three Django subsystems talk to Redis:
1. **Django cache** — session data, M-PESA access tokens, rate limit counters
2. **Celery broker** — task queue for async work
3. **Channels layer** — WebSocket message routing between workers

**Persistence:** Appendonly file (AOF) enabled for durability. Data loss on Redis crash would break in-flight tasks but not permanent state (all permanent data is in Postgres).

---

### Celery Workers + Beat

**Purpose:** Async task processing and scheduled jobs  
**Runs:** 3 worker processes + 1 beat process (scheduler)

**Worker concurrency:** 4 threads per worker (total 12 concurrent tasks)

**Task categories:**
- Financial reconciliation (daily, critical)
- Notification dispatch (async, retriable)
- Cleanup jobs (stale media, expired sessions)
- Scheduled escrow releases

**Beat schedule (key entries):**
- `reconcile_wallet_balances` — daily at 02:00 UTC
- `reset_daily_withdrawal_limits` — midnight UTC
- `process_scheduled_escrow_releases` — every 5 minutes
- `alert_on_dispatch_failures` — every 5 minutes

---

## Observability

**Logging:** Structured JSON logs to `stdout`, collected by Docker/systemd, forwarded to log aggregation

**Monitoring:** Lightweight self-hosted stack:
- **Prometheus** — metrics collection (Django exports metrics via `django-prometheus`)
- **Grafana** — dashboards (request rate, error rate, escrow state distribution, wallet balance totals)
- **Alertmanager** — alert routing to Slack

**Health checks:**
- `/health/` — Django responds 200 if DB and Redis are reachable
- M-PESA health check — periodic test requests to Daraja API to detect downtime before users do

---

## Deployment Process

**CI/CD:** GitHub Actions

1. On push to `main`: run tests, build Docker image, push to registry
2. SSH to production server
3. `docker-compose pull` → fetch new image
4. `docker-compose up -d` → rolling restart (zero-downtime via Caddy health checks)
5. Run migrations if needed: `docker exec zaruni-django python manage.py migrate`

**Rollback:** Re-deploy previous image tag, run reverse migrations if schema changed

---

## Scaling Path

Current bottleneck is **not** compute — it's operational complexity. The single VPS handles the current load with CPU usage <30% and memory <50%.

**When we'll scale:**
1. **Database read replicas** — when read-heavy queries (product search, order history) slow down writes
2. **Separate chat service** — if WebSocket connections dominate resource usage
3. **Horizontal Django workers** — add a second app server when request rate exceeds 1000 req/s sustained
4. **Managed cloud database** — when the cost of managing Postgres backups, failover, and monitoring exceeds the cost of RDS/managed Postgres

Not before. Premature scaling adds operational burden without user-visible benefit.

---

*← Back to [README](../README.md)*
