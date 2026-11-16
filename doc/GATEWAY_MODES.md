# Gateway Modes

This document describes the gateway layer of Mesh Router.

> **Cloudflare Worker retired.** This page used to compare three modes: "Cloudflare Worker
> Only", "OpenResty Gateway Only" and "CF Worker + Gateway Fallback". The Worker
> (`mesh-router-gateway-cf`, now an archived repo) was retired on staging (`inojob.com`)
> 2026-11-09 and on prod (`nsl.sh`) 2026-11-16, together with its `*.<zone>/*` route, its
> `gateway.entrypoint` fallback name and its `RESOLVE_URL` / `GATEWAY_FALLBACK_URL` settings.
> The OpenResty gateway is the only mode left.

> **nip.io replaced (2026-10-23).** Where this page says nip.io, a box with `IP_ZONE` uses
> `<ip-dashed>.<IP_ZONE>` (our nameserver, `*.<IP_ZONE>` in the box certificate) instead; nip.io
> remains only on a box without `IP_ZONE`. See the note atop [DATA_FLOW.md](DATA_FLOW.md).

---

## Overview

| Mode | Description | Status |
|------|-------------|--------|
| **OpenResty Gateway** | Self-hosted reverse proxy (`mesh-router-gateway`), one or several behind DNS | Current, staging and prod |
| ~~Cloudflare Worker Only~~ | Edge routing via a CF Worker | Retired 2026-11-09 staging, 2026-11-16 prod |
| ~~CF Worker + Gateway Fallback~~ | CF Worker with OpenResty fallback | Retired with the Worker |

---

## OpenResty Gateway

Our nameservers answer every name of the zone. A dash name (`<app>-<user>.<zone>`) resolves to
every healthy gateway (prod: 151.115.97.163 and 51.15.79.11). The gateway terminates TLS with
its own ZeroSSL `*.<zone>` certificate, resolves the box's routes from mesh-router-backend and
proxies to the box directly (agent route) or through the WireGuard tunnel (tunnel route). A
gateway without a route of its own to a NAT box hands the request to the gateway that has one
(see "Handoff between gateways" in the gateway README). Dot names of a live user zone resolve
straight to the box and do not go through a gateway.

```
┌──────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────┐
│  Client  │────▶│  Our DNS        │     │   OpenResty     │────▶│   PCS   │
│          │     │  (nsp1/nsp2)    │     │   Gateway(s)    │     │         │
└────┬─────┘     └─────────────────┘     │  (Nginx + Lua)  │     └─────────┘
     │                                   └─────────────────┘
     └──────────── HTTPS ───────────────────────▲  │
                                                   ▼
                                         Lua resolver queries
                                         mesh-router-backend
```

### Configuration

```bash
# Environment variables (full list in the gateway README)
SERVER_DOMAIN=nsl.sh
BACKEND_URL=http://backend:8192
CACHE_TTL=60
GATEWAY_FORWARDED_FOR=replace   # the gateway is the edge since the Worker's retirement
```

### Requirements

- Server(s) to host the gateway, listed in the backend's `DNS_GATEWAY_IPS`
- mesh-router-backend running and accessible
- A public `*.<zone>` certificate (`GATEWAY_ACME_*`, DNS-01 through our nameservers)

### Performance (historical)

Measured while the Cloudflare CDN still sat in front of the gateway:

| Metric | Value |
|--------|-------|
| Throughput | ~159 req/s |
| p50 Latency | ~283ms |
| p95 Latency | ~517ms |

**Note**: Higher latency due to no SSL session reuse between gateway and PCS (architectural limitation of dynamic `proxy_pass`).

For comparison, the retired CF Worker direct path measured ~249 req/s, ~169ms p50, ~246ms p95.

---

## Related Documentation

- [Data Flow](./DATA_FLOW.md) - Detailed request flow diagrams
- [Security in Transit](./SECURITY_IN_TRANSIT.md) - Certificate and encryption details
- [OpenResty Gateway README](../mesh-router-gateway/README.md)
