# Zaruni Technology Stack

This document explains our technology choices and the reasoning behind them.

**Design Philosophy:** Choose boring, proven technology. Optimize for maintainability and team velocity over theoretical peak performance.

---

## Backend Technologies

### Django + Django REST Framework

**Why we chose it:**
- Excellent ORM with migration system
- Batteries-included approach speeds development
- Strong security defaults (CSRF, XSS, SQL injection protection)
- Large ecosystem of reusable packages
- Python's readability makes onboarding easier

**Alternative considered:** FastAPI  
**Tradeoff:** Slightly less performance, but faster development and better for complex business logic. FastAPI is better for pure API microservices; Django is better for full-featured applications.

**Use in Zaruni:** Core API server, admin interface, business logic layer.

---

### PostgreSQL

**Why we chose it:**
- ACID compliance for financial transactions
- JSONB support for flexible schema evolution
- Better transaction guarantees than MySQL
- Excellent full-text search built-in
- Strong community and tooling

**Alternative considered:** MySQL  
**Tradeoff:** Slightly steeper learning curve, but superior reliability for escrow/wallet operations where consistency matters.

**Use in Zaruni:** Primary database for all transactional data, escrow state, user accounts.

---

### PgBouncer

**Why we chose it:**
- Connection pooling reduces PostgreSQL resource consumption
- Handles connection spikes without exhausting database connections
- Lightweight and battle-tested

**Alternative considered:** Built-in Django connection pooling  
**Tradeoff:** Extra component to manage, but essential at scale for efficient connection reuse.

**Use in Zaruni:** Connection pool between Django and PostgreSQL.

---

### Redis

**Why we chose it:**
- Fast in-memory cache (sub-millisecond latency)
- Excellent Celery broker (task queue)
- Session storage
- Rate limiting with atomic operations

**Alternative considered:** Memcached  
**Tradeoff:** More features than Memcached, slightly more complex, but worth it for Celery integration.

**Use in Zaruni:** Cache layer, Celery broker, rate limiting, session storage.

---

### Celery + Celery Beat

**Why we chose it:**
- Robust async task processing
- Retry logic and error handling built-in
- Scheduled jobs with Celery Beat
- Excellent monitoring tools

**Alternative considered:** Django Q, RQ  
**Tradeoff:** More complex setup, but industry standard with best monitoring and reliability.

**Use in Zaruni:**
- Payment reconciliation (scheduled)
- Notification dispatch (async)
- Cleanup jobs (scheduled)
- Escrow auto-release (scheduled)

---

### Django Channels

**Why we chose it:**
- WebSocket support within Django ecosystem
- Maintains same auth/permission model
- Integrates with existing middleware

**Alternative considered:** Separate Node.js service  
**Tradeoff:** Not as performant as dedicated Node/Socket.io, but simpler deployment and unified codebase.

**Use in Zaruni:** Real-time chat, live notifications, delivery tracking.

---

## Frontend Technologies

### React Native + Expo

**Why we chose it:**
- Single codebase for iOS and Android
- JavaScript ecosystem (large talent pool)
- Over-the-air updates with Expo
- Good performance for our use case
- Excellent developer experience

**Alternative considered:** Flutter  
**Tradeoff:** JavaScript/TypeScript over Dart. Chose React Native for larger ecosystem and easier hiring.

**Use in Zaruni:** Primary mobile application for all user-facing features.

---

### React + TypeScript (Admin Dashboard)

**Why we chose it:**
- Type safety catches errors at compile time
- Large component ecosystem
- Excellent tooling (VS Code, ESLint, Prettier)
- Team familiarity

**Alternative considered:** Vue.js  
**Tradeoff:** React has larger ecosystem and better TypeScript support.

**Use in Zaruni:** Admin/staff dashboard for operations, moderation, support.

---

### Astro (Marketing Site)

**Why we chose it:**
- Static site generation (fast page loads)
- SEO-friendly out of the box
- Low JavaScript payload
- Can integrate components from React/Vue/Svelte

**Alternative considered:** Next.js  
**Tradeoff:** Less flexible for apps, but superior for static content. Marketing site doesn't need app features.

**Use in Zaruni:** Public marketing website (zaruni.co.ke).

---

## Infrastructure

### Caddy

**Why we chose it:**
- Automatic HTTPS certificate management
- Zero-config certificate renewal
- Simpler configuration than Nginx
- Removes entire class of operational failures (expired certs)

**Alternative considered:** Nginx  
**Tradeoff:** Smaller community, but operational simplicity is worth it. Nginx is better at extreme scale; Caddy is better for small teams.

**Use in Zaruni:** Reverse proxy, HTTPS termination, static file serving.

---

### Docker + Docker Compose

**Why we chose it:**
- Consistent environments (dev/staging/prod)
- Easy service orchestration
- Good for monolith-first architecture
- Industry standard tooling

**Alternative considered:** Kubernetes  
**Tradeoff:** Kubernetes is overkill at current scale. Docker Compose is simpler and sufficient.

**Use in Zaruni:** All services containerized (Django, Postgres, Redis, Celery, Channels).

---

### Hetzner VPS

**Why we chose it:**
- 80% cost savings vs AWS/GCP
- Sufficient performance for current scale
- Full control over infrastructure
- Predictable pricing

**Alternative considered:** AWS  
**Tradeoff:** More operational burden (no managed services), but cost savings are significant at early stage. Will reconsider at larger scale.

**Use in Zaruni:** All production infrastructure.

---

## Development & Testing

### pytest + factory_boy

**Why we chose it:**
- Best Django test integration
- Fixture factories make test data management easy
- Parameterized tests
- Clear assertion messages

**Alternative considered:** Django's unittest  
**Tradeoff:** Slightly different syntax, but much more powerful.

---

### GitHub Actions

**Why we chose it:**
- Free for public repos
- Good Docker integration
- Simple YAML configuration
- Integrated with GitHub

**Alternative considered:** GitLab CI  
**Tradeoff:** Chose GitHub for platform familiarity.

---

### Playwright (E2E Testing)

**Why we chose it:**
- Cross-browser support
- Auto-wait (no flaky tests)
- Good debugging tools
- Modern API

**Alternative considered:** Cypress  
**Tradeoff:** Playwright has better multi-browser support and faster execution.

---

## Decisions We'd Revisit at Scale

Some choices work well now but would change at 10x-100x scale:

1. **Self-hosted → Managed services**: Would move to AWS RDS, ElastiCache, etc. for better reliability SLAs
2. **Docker Compose → Kubernetes**: Would orchestrate with K8s for auto-scaling
3. **Monolith → Microservices**: Would extract high-traffic modules (payments, chat) into separate services
4. **Single region → Multi-region**: Would deploy closer to users for lower latency

But premature optimization is the root of all evil. These are future problems, not current ones.

---

**Last Updated:** July 2026
