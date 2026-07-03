# ADR-0002: Caddy over Nginx

**Status:** Accepted  
**Date:** Early 2025  
**Context:** Zaruni needed a reverse proxy for a self-hosted production stack operated by a small team. The proxy handles HTTPS termination, routing to backend services, and static file serving.

---

## Decision

We use Caddy as our reverse proxy.

## Reasoning

The single biggest operational risk on a self-hosted stack is an expired TLS certificate. An expired cert takes the entire platform offline and is the kind of failure that happens at the worst possible time — because it's calendar-driven, not load-driven, so it doesn't show up in load tests.

Caddy eliminates this class of failure entirely. Certificate acquisition and renewal happen automatically via ACME (Let's Encrypt). There is no cron job to write, no certbot to maintain, no `systemctl reload nginx` to script into a renewal hook. The cert renews itself. If it fails to renew, Caddy retries. We have not manually touched a TLS certificate since deploying Caddy.

The second reason is configuration complexity. A production-grade Nginx config with correct security headers, WebSocket proxying for Django Channels, gzip, and static file handling is several hundred lines and requires understanding a non-obvious directive order. The equivalent Caddyfile is around 40 lines and reads like prose:

```
zaruni.co.ke {
    reverse_proxy /ws/* django:8000
    reverse_proxy /api/* django:8000
    file_server /static/* {
        root /srv/static
    }
}
```

For a small team, configuration that fits in your head is safer than configuration that requires a reference manual.

## Alternatives Considered

**Nginx**  
The obvious choice — enormous community, every StackOverflow answer assumes it, battle-tested at scales far beyond ours. The argument against wasn't performance (Nginx is faster at extreme concurrency) but operational simplicity. Nginx does not manage certificates automatically. At our scale, the cert management overhead outweighs the performance advantage.

**Traefik**  
Also supports automatic HTTPS and has good Docker integration. We chose Caddy because its configuration model is simpler for our use case (no dynamic service discovery needed — our stack is static Docker Compose services).

**AWS ALB / Cloudflare**  
Considered routing through a managed proxy. Rejected at this stage because it adds a paid dependency and latency hop for a stack that doesn't need global distribution yet.

## Consequences

**Better:**
- Zero certificate expiry incidents since deployment
- Simpler config that the whole team can read and modify
- WebSocket proxying for Django Channels works out of the box
- HTTP/2 enabled by default

**Harder:**
- Smaller community than Nginx — StackOverflow answers often need translating
- Some advanced config patterns (complex rate limiting, Lua scripting) require Nginx or a separate layer
- Less battle-tested at very high concurrency (10k+ req/s) — not a current concern

**Accepted tradeoff:**  
Community surface area matters less than operational simplicity for a small team. We can always put Nginx in front if we hit Caddy's limits — we have not come close.

---

*← Back to [decisions](../README.md#️-architecture-decision-records)*
