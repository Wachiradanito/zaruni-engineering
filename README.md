# Zaruni Engineering

> Architecture, case studies, and extracted tools from building a multi-service super-app for Kenya

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

## What is Zaruni?

Zaruni is a multi-service marketplace and jobs platform handling real M-PESA payments at production scale in Kenya. It combines:

- 💼 **Jobs Marketplace** — Gig worker matching and task management
- 🛠️ **Service Bookings** — Home services, repairs, and professional services
- 🏪 **Product Marketplace** — Local shops and C2C commerce
- 🚚 **Delivery Orchestration** — Logistics and last-mile delivery
- 💬 **Real-time Chat** — In-app messaging for all transactions
- 💰 **Escrow-backed Payments** — M-PESA integration with payment guarantees

**Live Product:** [zaruni.co.ke](https://zaruni.co.ke)

## Why This Repository Exists

Zaruni's source code is private to protect the business. This repository documents the architecture, shares engineering decisions, and showcases the reusable components we've extracted and open-sourced.

Think of this as the **engineering museum exhibit** — the real artifacts remain in the archive, but you can see what we built and how we built it.

### The Two-Repo Model

| `zaruni` (private) | `zaruni-engineering` (public) |
|-------------------|-------------------------------|
| Actual source code | Architecture narratives |
| Business logic | Tech decision documentation |
| Production data models | Problem-solving case studies |
| **For:** Running the product | **For:** Proving engineering competence |

This is how companies like Twitter, GoDaddy, and Red Hat operate — closed source product, open engineering insights.

## What's in This Repository

### 📐 Architecture Documentation
System design, data flow diagrams, and module interaction patterns.

- [System Overview](architecture/system-overview.md) — *Coming Q3 2026*
- [Payments Architecture](architecture/payments-architecture.md) — *Coming Q3 2026*
- [Authentication System](architecture/auth-architecture.md) — *Coming Q3 2026*
- [Deployment Topology](architecture/deployment-topology.md) — *Coming Q3 2026*

### 📖 Case Studies
Real engineering problems we solved, how we investigated them, and the solutions we built.

- [M-PESA Callback Idempotency](case-studies/2025-mpesa-idempotency.md) — *Coming Soon*
- [Escrow State Machine Design](case-studies/2025-escrow-design.md) — *Coming Soon*
- [Verification Tier System](case-studies/2025-verification-tiers.md) — *Coming Soon*

### ⚖️ Architecture Decision Records
Why we chose specific technologies and approaches (ADRs).

- [ADR-0001: Postgres over MySQL](decisions/0001-postgres-over-mysql.md) — *Coming Soon*
- [ADR-0002: Caddy over Nginx](decisions/0002-caddy-over-nginx.md) — *Coming Soon*
- [ADR-0003: Self-hosted vs Cloud](decisions/0003-self-hosted-vs-cloud.md) — *Coming Soon*
- [ADR-0004: Monolith-first Architecture](decisions/0004-monolith-first.md) — *Coming Soon*

### 📦 Extracted Open-Source Tools
Reusable packages we've built and open-sourced — zero Zaruni-specific business logic.

See [extracted-tools/README.md](extracted-tools/README.md) for the full list and links.

**Available Now:**
- **django-mpesa** — M-PESA payment integration for Django

**In Development:**
- **mainauth** — Phone-based authentication with OTP
- **django-notify** — Multi-channel notification system

### 🛠️ Technology Stack
See [stack.md](stack.md) for the complete tech stack with reasoning for each choice.

## Technology Stack (Summary)

**Backend:**
- Django + Django REST Framework — Rapid development, excellent ORM
- PostgreSQL + PgBouncer — ACID compliance, connection pooling
- Redis — Cache + Celery broker
- Celery + Beat — Async tasks + scheduled jobs
- Django Channels — WebSocket support for real-time chat

**Frontend:**
- React Native + Expo — Cross-platform mobile (iOS/Android)
- React + TypeScript — Admin dashboard
- Astro — Marketing site (static generation)

**Infrastructure:**
- Caddy — Reverse proxy with automatic HTTPS
- Gunicorn — WSGI/ASGI application server
- Docker + Docker Compose — Container orchestration
- Self-hosted on Hetzner VPS — Cost optimization

See [stack.md](stack.md) for detailed reasoning behind each choice.

## Key Design Principles

1. **Modular monolith first** — Clear module boundaries enable future microservices extraction
2. **Event-driven architecture** — Async events for cross-module communication, direct imports for core logic
3. **Escrow-backed payments** — No direct transfers, always through state machine
4. **Progressive trust system** — Verification tiers unlock capabilities as users prove reliability
5. **Mobile-first design** — Web is secondary, mobile app is primary interface
6. **Safety by design** — Rate limiting, idempotency, transaction guards built-in

## Project Status

Zaruni is in **active development** with:
- ✅ Production deployment serving real users in Kenya
- ✅ M-PESA payment integration live and processing transactions
- ✅ Multi-module marketplace operational (Jobs, Services, Products, Delivery)
- ✅ Real-time chat system deployed
- 🚧 Continuous feature iteration and scaling optimizations

## How to Use This Repository

**For Recruiters:**  
Read case studies to understand problem-solving approach and system design thinking.

**For Engineers:**  
Check out ADRs for technology decisions. Study architecture docs for patterns applicable to your projects.

**For Learners:**  
Use this as a reference for building production-scale super-apps in emerging markets.

**For Open-Source Users:**  
Use our extracted tools in your own Django projects. They're battle-tested in production.

## About

**Daniel Maina / Mainfinity**  
Building Zaruni to solve marketplace trust and logistics challenges in Kenya.

- 🌐 Website: [zaruni.co.ke](https://zaruni.co.ke)
- 💼 LinkedIn: [linkedin.com/in/daniel-maina-mainfinity](https://linkedin.com/in/daniel-maina-mainfinity)
- 📧 Contact: Through zaruni.co.ke

## License

Documentation in this repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

Code snippets in case studies are MIT licensed unless otherwise specified.

Extracted packages have their own licenses — see individual repositories.

---

**Status:** 🚧 Phase 0 complete — documentation in progress

**Last Updated:** July 2026
