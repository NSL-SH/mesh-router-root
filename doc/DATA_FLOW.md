# Data Flow

This document provides detailed diagrams showing how data flows through the gateway in Mesh Router.

> **Cloudflare Worker retired (staging 2026-11-09, prod 2026-11-16).** The OpenResty gateway
> is the only gateway mode. Our nameservers answer every name: dash names resolve to the
> gateways, which terminate TLS with their own ZeroSSL `*.<zone>` and proxy to the box direct
> (agent) or through the WireGuard tunnel; dot names of a live user zone resolve straight to the
> box. The diagram below still shows Cloudflare in front of the gateway and its latencies were
> measured that way: both are historical. The former "Mode 2: Cloudflare Worker" and "Mode 3:
> CF Worker + Fallback" flows were removed with the Worker.

> **Route model changed (2026-10-23).** The diagrams below show the older six-route
> model (raw `ip`, sslip.io and nip.io agent routes). Today the agent registers one
> domain route per scheme: `<ip-dashed>.<IP_ZONE>` (agent 1.1.3+, our nameserver), or
> `<ip-dashed>.nip.io` on a box without `IP_ZONE`. The gateway connects by the route's `ip`
> (SNI: the box's hop name `<label>.box.<zone>`). Current table: [architecture/README.md](architecture/README.md#route-types).

---

## Request Flow Overview

All gateway modes follow a similar high-level pattern:

```
Client Request
      │
      ▼
┌─────────────┐
│  DNS        │  Resolves dash names to the gateways
│  Resolution │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Gateway    │  Routes request to correct PCS
│  Layer      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  PCS        │  Serves the application
│  Instance   │
└─────────────┘
```

---

## Mode 1: OpenResty Gateway Flow

### Complete Request Lifecycle

```
 CLIENT                    CLOUDFLARE              OPENRESTY GATEWAY                    BACKEND API                PCS
    │                          │                          │                                  │                      │
    │  1. HTTPS Request        │                          │                                  │                      │
    │  app.alice.nsl.sh/api    │                          │                                  │                      │
    │─────────────────────────▶│                          │                                  │                      │
    │                          │                          │                                  │                      │
    │                          │  2. Forward to origin    │                                  │                      │
    │                          │─────────────────────────▶│                                  │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  3. Lua: Extract subdomain       │                      │
    │                          │                          │     "alice" from Host header     │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  4. Check local cache            │                      │
    │                          │                          │     (10MB shared dict)           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  5. Cache MISS                   │                      │
    │                          │                          │─────────────────────────────────▶│                      │
    │                          │                          │  GET /resolve/v2/alice           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  6. Return routes                │                      │
    │                          │                          │◀─────────────────────────────────│                      │
    │                          │                          │  {routes: [{ip, port, priority}]}│                      │
    │                          │                          │                                  │                      │
    │                          │                          │  7. Select best route            │                      │
    │                          │                          │     (lowest priority, healthy)   │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  8. NEW TLS Handshake            │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  (100-120ms overhead)            │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  9. Proxy request                │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  Headers: Host, X-Real-IP,       │                      │
    │                          │                          │  X-Forwarded-For/Proto           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  10. Response from app           │                      │
    │                          │                          │◀─────────────────────────────────────────────────────────│
    │                          │                          │                                  │                      │
    │                          │  11. Return response     │                                  │                      │
    │                          │◀─────────────────────────│                                  │                      │
    │                          │                          │                                  │                      │
    │  12. Response            │                          │                                  │                      │
    │◀─────────────────────────│                          │                                  │                      │
    │                          │                          │                                  │                      │

Total latency: ~283ms (p50)
├── Client → CF: ~30ms
├── CF → Gateway: ~20ms
├── Resolver lookup: <1ms (cache) or ~50ms (API call)
├── Gateway → PCS TLS: ~100-120ms  ◀── MAIN BOTTLENECK (no session reuse)
├── PCS processing: ~30ms
└── Return path: ~50ms
```

### Lua Resolver Logic

```
┌─────────────────────────────────────────────────────────────┐
│                    Lua Resolver (access phase)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Parse Host Header                                       │
│     ┌─────────────────────────────────────────────────┐     │
│     │ app.alice.nsl.sh → subdomain = "alice"         │     │
│     │ alice.nsl.sh     → subdomain = "alice"         │     │
│     │ nsl.sh           → subdomain = nil (default)   │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│                           ▼                                 │
│  2. Check Cache                                             │
│     ┌─────────────────────────────────────────────────┐     │
│     │ shared_dict "resolver_cache" (10MB)            │     │
│     │ Key: "alice"                                   │     │
│     │ TTL: 60-300 seconds                            │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│              ┌────────────┴────────────┐                    │
│              │                         │                    │
│           HIT                        MISS                   │
│              │                         │                    │
│              ▼                         ▼                    │
│     Use cached routes          Query Backend API            │
│                                GET /resolve/v2/alice        │
│                                        │                    │
│                                        ▼                    │
│                               Store in cache                │
│                                        │                    │
│              └────────────┬────────────┘                    │
│                           │                                 │
│                           ▼                                 │
│  3. Route Selection                                         │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Routes sorted by priority (1 = best)           │     │
│     │                                                 │     │
│     │ For each route:                                │     │
│     │   ├── No healthCheck? → Use immediately       │     │
│     │   └── Has healthCheck?                        │     │
│     │       ├── Check health cache (5 min TTL)      │     │
│     │       ├── If expired → HTTP HEAD probe        │     │
│     │       └── If healthy → Use this route         │     │
│     │                                                 │     │
│     │ All unhealthy? → Use first route as fallback  │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│                           ▼                                 │
│  4. Set Backend Variable                                    │
│     ngx.var.backend = "https://[ip]:port"                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Retired: Cloudflare Worker Flows

The Cloudflare Worker (`mesh-router-gateway-cf`) fetched the box by an IP-encoded name and fell
back to the OpenResty gateway through `gateway.entrypoint.<zone>`. Both flows ended with the
Worker's retirement (staging 2026-11-09, prod 2026-11-16): the Worker, its `*.<zone>/*` route
and the `gateway.entrypoint` record are deleted. IP-encoded names (`ip.<zone>`) remain, as the
agent route domain.

---

## Route Types

Routes are categorized by `type` field and selected based on the routing component:

```
┌─────────────────────────────────────────────────────────────┐
│                       ROUTE TYPES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  IP ROUTES (type: "ip")                                     │
│  ─────────────────────────────────────                      │
│  Used by: Gateway (connects to IPs directly)                │
│                                                             │
│  ┌──────────┬──────────┬────────────────────────────────┐   │
│  │ Priority │ Source   │ Example                        │   │
│  ├──────────┼──────────┼────────────────────────────────┤   │
│  │ 1        │ agent    │ 88.187.147.189:443 (public)    │   │
│  │ 10       │ tunnel   │ 172.30.0.3:80 (WireGuard)      │   │
│  └──────────┴──────────┴────────────────────────────────┘   │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  DOMAIN ROUTES (type: "domain")                             │
│  ─────────────────────────────────────                      │
│  Used by: the retired CF Worker (could not connect to IPs)  │
│                                                             │
│  ┌──────────┬──────────┬────────────────────────────────┐   │
│  │ Priority │ Source   │ Example                        │   │
│  ├──────────┼──────────┼────────────────────────────────┤   │
│  │ 2        │ agent    │ 88-187-147-189.sslip.io:443    │   │
│  │ 3        │ agent    │ 88-187-147-189.nip.io:443      │   │
│  └──────────┴──────────┴────────────────────────────────┘   │
│                                                             │
│  Domain routes use wildcard DNS services (sslip.io, nip.io) │
│  that resolve embedded IPs back to the actual IP address.   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Route selection: Lower priority number wins.
The gateway connects by `ip` for every route; the domain was for the retired CF Worker.
```

## Route Registration & Validation

Routes are validated at registration time before being stored:

```
┌─────────────────────────────────────────────────────────────┐
│                  ROUTE VALIDATION FLOW                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Agent submits routes to backend:                           │
│  POST /routes/:userid/:sig                                  │
│  {                                                          │
│    "routes": [                                              │
│      {"type": "ip", "ip": "88.187.147.189", "port": 443},  │
│      {"type": "domain", "domain": "88-187-147-189.sslip.io"}│
│      {"type": "domain", "domain": "88-187-147-189.nip.io"} │
│    ]                                                        │
│  }                                                          │
│                           │                                 │
│                           ▼                                 │
│  Backend validates each route:                              │
│                                                             │
│  For each route:                                            │
│  ├─ Build test URL:                                         │
│  │   IP route:     https://88.187.147.189:443/health        │
│  │   Domain route: https://88-187-147-189.sslip.io/health   │
│  │                                                          │
│  ├─ Test connectivity (5 second timeout)                    │
│  │   ├─ Connection refused? → Reject                        │
│  │   ├─ Timeout? → Reject                                   │
│  │   └─ Connected? → Continue                               │
│  │                                                          │
│  └─ Verify SSL (if HTTPS)                                   │
│      ├─ Cert valid? → Accept                                │
│      └─ Cert invalid? → Reject                              │
│                           │                                 │
│                           ▼                                 │
│  Store only healthy routes in Redis                         │
│  Return: { accepted: [...], rejected: [...] }               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Benefits:
• Bad routes never reach Redis → No timeout delays
• nip.io/sslip.io outages detected at registration
• Agent knows which routes failed and why
```

---

## Health Check Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    HEALTH CHECK FLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Route Configuration:                                       │
│  {                                                          │
│    "ip": "203.0.113.5",                                    │
│    "port": 443,                                            │
│    "priority": 1,                                          │
│    "healthCheck": {                                        │
│      "path": "/.well-known/health",                        │
│      "host": "alice.nsl.sh"    // optional                 │
│    }                                                       │
│  }                                                          │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  Active Health Check (before routing):                      │
│                                                             │
│    1. Check health cache (TTL: 5 minutes)                   │
│                                                             │
│       Cache HIT & healthy    → Use route                    │
│       Cache HIT & unhealthy  → Skip route                   │
│       Cache MISS             → Probe endpoint               │
│                                                             │
│    2. HTTP HEAD probe                                       │
│       HEAD https://ip:port/.well-known/health               │
│       Timeout: 2 seconds                                    │
│                                                             │
│       Status 200  → Mark healthy, cache 5 min               │
│       Other       → Mark unhealthy, cache 5 min             │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  Passive Health Tracking (after routing):                   │
│                                                             │
│    Gateway tracks proxy failures per route.                 │
│                                                             │
│    Failure (502, timeout, connection refused):              │
│      → Increment failure counter                            │
│      → 3 consecutive failures → Mark unhealthy 1 min        │
│                                                             │
│    Success (any 2xx-5xx from PCS):                          │
│      → Reset failure counter                                │
│      → Mark healthy                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Documentation

- [Gateway Modes](./GATEWAY_MODES.md) - Comparison and decision tree
- [Security in Transit](./SECURITY_IN_TRANSIT.md) - Certificate and encryption details
- [Routing Architecture](../mesh-router-gateway/doc/ROUTING_ARCHITECTURE.md) - Multi-route failover details
