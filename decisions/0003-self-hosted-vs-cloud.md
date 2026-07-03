# ADR-0003: Self-hosted on Hetzner over Managed Cloud (AWS/GCP)

**Status:** Accepted  
**Date:** Early 2025  
**Context:** Zaruni needed production infrastructure. The choice was between managed cloud services (AWS, GCP, Azure) and renting dedicated/VPS servers and managing the stack ourselves.

---

## Decision

We self-host on Hetzner VPS servers rather than using managed cloud services.

## Reasoning

The core question was: what does managed cloud actually buy us at our current scale, and what does it cost?

At early-stage scale, AWS buys you:
- Managed database (RDS) — no manual Postgres updates or backups to script
- Managed cache (ElastiCache) — no Redis configuration
- Auto-scaling — handles traffic spikes automatically
- Global CDN — low latency worldwide
- 99.99% SLA — formal uptime guarantees

What it costs:
- RDS `db.t3.medium` + ElastiCache + ALB + EC2 baseline ≈ **$300-500/month minimum**
- That's before storage, bandwidth, or any growth in traffic

What Hetzner costs for equivalent compute:
- A `CPX31` (4 vCPU, 8GB RAM) with 160GB NVMe: **€15.90/month**
- A `CPX21` for staging: **€7.90/month**

At pre-revenue or early-revenue stage, the difference between €25/month and $400/month is the difference between running sustainably and burning runway on infrastructure that isn't the bottleneck.

The tradeoff is operational burden. With Hetzner, we manage:
- Postgres backups (scripted to object storage)
- Redis configuration
- Certificate renewal (solved by Caddy — see ADR-0002)
- OS updates and security patches

This is real work. But it's a one-time setup cost, not a recurring one. The scripts exist, the backups run, the monitoring alerts. It took roughly a week to set up correctly and requires maybe 30 minutes of attention per month.

The honest assessment: AWS earns its premium when you need its features — global distribution, auto-scaling to handle 10x traffic spikes, compliance certifications, SLA-backed uptime. We don't need any of those yet. When we do, migrating is straightforward because we containerised everything with Docker from day one.

## Alternatives Considered

**AWS (EC2 + RDS + ElastiCache)**  
The "right" choice at scale. Wrong choice at our current stage purely on unit economics. Revisit when monthly infrastructure spend is less than 1% of revenue.

**DigitalOcean / Render / Railway**  
Good middle ground — managed Postgres, easier deploys, reasonable pricing. Chose Hetzner for lower base cost and because we wanted to learn the full stack rather than abstract it away.

**Fly.io**  
Interesting for global distribution. Adds complexity we don't need yet.

**Supabase (managed Postgres)**  
Strong option for the database layer specifically. Chose self-managed Postgres because we use `select_for_update(skip_locked=True)` and some Postgres-specific features that need version control, and we wanted full visibility into database configuration.

## Consequences

**Better:**
- Infrastructure cost is ~5% of what equivalent AWS setup would cost
- Full control over configuration, Postgres settings, Redis tuning
- No vendor lock-in — Docker Compose means moving providers is straightforward
- Forces understanding of the full stack (operationally valuable)

**Harder:**
- Responsible for our own backups, updates, and incident response
- No auto-scaling — a sudden spike requires manual intervention
- No formal SLA — Hetzner has good uptime in practice but no contractual guarantee
- More setup time upfront

**Accepted tradeoff:**  
Operational burden is acceptable at current scale. We have monitoring, automated backups, and documented runbooks. We will revisit this when the cost of engineer time managing infrastructure exceeds the savings over managed cloud.

---

*← Back to [decisions](../README.md#️-architecture-decision-records)*
