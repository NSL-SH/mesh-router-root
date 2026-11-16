# Mesh Router Architecture

This document describes the overall architecture of the Mesh Router system.

## Request Flow

```
                         External Request
                                │
                                ▼
                     ┌──────────────────────┐
                     │  Our nameservers     │  dash name → gateways
                     │  (PowerDNS, nsp1/2)  │  dot name of a live user → the box
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │  mesh-router-gateway │  ZeroSSL *.<zone>, resolve,
                     │  (OpenResty, 1..n)   │  handoff between gateways
                     └──────────┬───────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
   ┌──────────────────────┐            ┌──────────────────────┐
   │  Agent route         │            │  Tunnel route        │
   │  (direct, prio 1)    │            │  (WireGuard, prio 2) │
   └──────────┬───────────┘            └──────────┬───────────┘
              └─────────────────┬─────────────────┘
                                ▼
                     ┌──────────────────────┐
                     │    User's PCS        │
                     │  (Caddy → Services)  │
                     └──────────────────────┘
```

The Cloudflare Worker (`mesh-router-gateway-cf`, edge routing with a gateway fallback) that
used to sit in front was retired on staging 2026-11-09 and on prod 2026-11-16, after the zones'
NS moved to our nameservers (2026-11-09 / 2026-11-14).

## Route Types

Routes are categorized by type and selected by priority (lower number = higher priority):

| Type | Priority | Source | Example | Used By |
|------|----------|--------|---------|---------|
| `domain` | 1 | agent | `88-187-147-189.ip.nsl.sh` (ip `88.187.147.189`), https + http | Gateway (OpenResty) |
| `ip` | 2 | tunnel | `172.30.0.3:80` (the tunnel server) | Gateway (via WireGuard) |

The agent's domain is `<ip-dashed>.<IP_ZONE>` when the box has `IP_ZONE` (agent 1.1.3+,
our nameserver), otherwise `<ip-dashed>.nip.io`.

### Why IP-encoded names?

- **Historical reason:** Cloudflare Workers cannot fetch IP addresses directly (error 1003),
  so the Worker fetched `<app>-<ip>.<IP_ZONE>`. The Worker is retired (2026-11-16); the
  `ip.<zone>` names stay, as the agent route domain.
- **The OpenResty gateway** connects to the route's `ip`, SNI the box's hop name
  `<label>.box.<zone>`

### Route Selection by Component

| Component | Route Types Used | Selection Logic |
|-----------|------------------|-----------------|
| **OpenResty Gateway** | all | Pick lowest priority healthy route (agent, then tunnel) |

## Route Registration & Validation

Routes are validated at registration time by the backend:

```
Agent                           Backend                         Redis
  │                                │                               │
  ├─ POST /routes                  │                               │
  │  { routes: [                   │                               │
  │    {type: "domain", domain: "<ip>.ip.nsl.sh", scheme: https},  │
  │    {type: "domain", domain: "<ip>.ip.nsl.sh", scheme: http}    │
  │  ]}                            │                               │
  │ ───────────────────────────────>                               │
  │                                │                               │
  │                    For each route:                             │
  │                    ├─ Test connectivity (5s timeout)           │
  │                    ├─ Verify SSL if HTTPS                      │
  │                    └─ Only store if healthy                    │
  │                                │                               │
  │                                ├───────────────────────────────>
  │                                │  Store healthy routes only    │
  │<───────────────────────────────┤                               │
  │  { accepted: [...], rejected: [...] }                          │
```

**Key Benefit**: Bad routes are never stored, eliminating timeout delays in the request path.

## Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **mesh-router-gateway** | Public edge: TLS termination, resolution, proxy to the box | OpenResty (Nginx + Lua) |
| **mesh-router-backend** | Domain/route registry, route validation | Express.js, Firebase, Redis |
| **mesh-router-tunnel** | WireGuard VPN for NAT traversal | TypeScript, WireGuard |
| **mesh-router-agent** | IP + domain route registration | TypeScript |
| **mesh-dashboard** | User management UI | Next.js 14, React Admin |
| **mesh-router-nameserver** | Authoritative DNS for the zones | PowerDNS |

## Data Flows

### Registration Flow

```
mesh-dashboard ─── Firebase Auth ───► Firebase
       │
       │ (user registers domain)
       ▼
mesh-router-backend ───► Firestore (domain data)
                    ───► Redis (route cache)
```

### Route Registration Flow

```
mesh-router-agent ─── POST /routes/:userid/:sig ───► mesh-router-backend
       │                                                     │
       │ Routes submitted:                                   │ Validation:
       │ ├─ {type: domain, <ip>.ip.nsl.sh, https, prio 1}   │ ├─ Test connectivity
       │ └─ {type: domain, <ip>.ip.nsl.sh, http, prio 1}    │ ├─ Verify SSL
       │                                                     │ └─ Store only healthy
       │                                                     ▼
       │                                                  Redis
       │                                               (healthy routes)
       │
mesh-router-tunnel ─── POST /routes/:userid/:sig ───► mesh-router-backend
       │                                                     │
       │ Routes submitted:                                   │
       │ └─ {type: ip, ip: 172.30.x.x, priority: 10}        │
       │                                                     ▼
       │                                                  Redis
```

### Request Resolution Flow

**OpenResty Gateway:**
```
OpenResty ─── GET /resolve/v2/:domain ───► mesh-router-backend
    │                                              │
    │                                              ▼
    │                                   Redis (route lookup)
    │                                              │
    │◄── {routes: [{type, ip/domain, port, priority}]} ──┘
    │
    ├─ Filter: request scheme (http/https)
    ├─ Sort by priority (lowest first)
    ├─ Check health status
    │
    ▼
 Proxy to: https://[route.ip]:{port}/
```

## Domain Examples

- `casaos-username.nsl.sh` - CasaOS container management
- `immich-username.nsl.sh` - Immich photo management
- `nextcloud-username.nsl.sh` - Nextcloud file sync
- Custom: `*.mydomain.com` (with custom domain setup)

## Security

- **Authentication**: Firebase Auth for user registration, Ed25519 signatures for route registration
- **TLS**: ZeroSSL `*.<zone>` on the gateways (Let's Encrypt backup); mesh-CA and Let's Encrypt certificates on the box (Caddy)
- **DDoS Protection**: no Cloudflare in front since the NS move; several gateways behind DNS
- **VPN**: WireGuard for encrypted tunnel traffic

## Deployment

For server-side deployment instructions including network configuration:

- [Server Deployment Guide](./SERVER_DEPLOYMENT.md) - Backend, gateway, tunnel, and dashboard deployment with mesh-internal network setup

## Diagram Files

Architecture diagrams are available as Excalidraw files:

| File | Description |
|------|-------------|
| `mesh-router/mesh-router-arch-v1.exdraw` | Legacy architecture diagram |
| `mesh-router/mesh-router-arch-v2.exdraw` | Current architecture diagram |

Open `.exdraw` files with [Excalidraw](https://excalidraw.com/) for interactive viewing and editing.
