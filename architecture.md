# Architecture

> **All addresses on this page are placeholders.** The LAN block `10.0.0.0/24` is
> illustrative and does not correspond to the real network. Hostnames are shown
> because they carry the design intent; the real domain is replaced with
> `example.com` throughout.

## Overview

Nine machines. Seven form a Docker Swarm that holds **no state at all**; the other
two hold everything that must survive.

```
                            ┌──────────────────────┐
                Internet ──▶│  Cloudflare          │
                            │  DNS · Tunnel · MFA  │
                            └──────────┬───────────┘
                                       │ outbound-only tunnel
                                       │ (no open inbound ports)
┌──────────────────────────────────────┼──────────────────────────────────┐
│  LAN 10.0.0.0/24                     │                                  │
│                                      ▼                                  │
│   ┌──────────────────────────────────────────────────────────────┐      │
│   │  DOCKER SWARM — stateless (7 nodes)                           │     │
│   │                                                               │     │
│   │   Managers (quorum = 3)          Workers (4)                  │     │
│   │   ┌──────────┐                   ┌────────────┐               │     │
│   │   │zs-node-01│ cloudflared       │zs-worker-01│               │     │
│   │   │zs-node-02│ socketproxy       │zs-worker-02│  dashboard    │     │
│   │   │zs-node-03│ (manager socket)  │zs-worker-03│  stirling-pdf │     │
│   │   └──────────┘                   │zs-worker-04│  glances      │     │
│   │                                  └────────────┘  (global)     │     │
│   │   No local volumes. Any task may run on any node.             │     │
│   └───────────┬───────────────────────────────┬───────────────────┘     │
│               │ NFS  /mnt/storage             │ TCP 5432                │
│               ▼                               ▼                         │
│   ┌───────────────────────┐      ┌──────────────────────────┐           │
│   │  zs-store-01          │      │  zs-state-01             │           │
│   │  NOT in the Swarm     │      │  NOT in the Swarm        │           │
│   │                       │      │                          │           │
│   │  NFS export           │      │  PostgreSQL 16           │           │
│   │  /srv/nfs/swarm-data  │      │  Portainer CE            │           │
│   │  → files & volumes    │      │  → databases & control   │           │
│   └───────────────────────┘      └──────────────────────────┘           │
│                                                                          │
│   Tailscale mesh across all nodes — management access, no open ports.    │
└──────────────────────────────────────────────────────────────────────────┘
```

## The nine machines

### Swarm cluster (7 nodes, fully stateless)

| Hostname | Role | Notes |
|---|---|---|
| `zs-node-01` | Manager | Swarm control plane; hosts the Docker socket proxy |
| `zs-node-02` | Manager | Swarm control plane |
| `zs-node-03` | Manager | Swarm control plane |
| `zs-worker-01` | Worker | Largest node — general workload, legacy NFS share |
| `zs-worker-02` | Worker | General workload |
| `zs-worker-03` | Worker | Runs the PDF toolkit |
| `zs-worker-04` | Worker | General workload (newest hardware) |

Three managers, because Raft quorum tolerates exactly one manager failure at three
**and also at four** — a fourth manager buys nothing. Everything else is a worker.

### Outside the Swarm (2 hosts)

| Hostname | Role | Runs |
|---|---|---|
| `zs-state-01` | Stateful host | PostgreSQL 16, Portainer CE |
| `zs-store-01` | Storage server | NFS export — the single storage backend for every stack |

The split between these two is deliberate: **`zs-state-01` runs stateful *services*
(a database engine, the control UI); `zs-store-01` holds *files*.** Different
failure modes, different backup needs, different hardware profiles.

## The central idea: the Swarm owns nothing

Docker Swarm does not replicate volumes. A container that writes to a local path is
married to the node it started on — and the moment that node dies, so does the data.
Most homelab Swarm setups paper over this by pinning services to nodes with
`placement.constraints`, which quietly turns a cluster back into a set of single
points of failure.

The approach here is to remove the reason to pin at all:

- **Files** live on `zs-store-01`, exported over NFS and mounted at `/mnt/storage`
  on *every* Swarm node. A service finds its data no matter where it starts.
- **Structured data** lives in PostgreSQL on `zs-state-01`, reached over the network.
- **Nothing else persists.** Anything a container writes locally is disposable.

The result: any Swarm node can be wiped and rebuilt from scratch without data loss,
and a service can be rescheduled anywhere. State is concentrated on two hosts that
can be backed up properly, instead of smeared across seven that cannot.

**A service may only be pinned to a node if it needs something genuinely node-local.**
Storage is no longer such a reason. What still qualifies:

- the Docker socket of a **manager** — only managers answer `/nodes`, `/services`
  and `/tasks`, which is why the socket proxy stays on one
- `mode: global` agents, which run everywhere by definition
- specific hardware attached to one host

See [`examples/nfs-volume-pattern.yml`](examples/nfs-volume-pattern.yml) for the
concrete pattern, including the failure mode it does *not* protect against.

## Ingress: a tunnel, not a reverse proxy

There is no Traefik, no nginx, no Caddy, and no port forwarding on the router.

A `cloudflared` daemon runs in the Swarm and opens an **outbound** connection to
Cloudflare. Public traffic arrives at Cloudflare and is pushed down that existing
connection. Consequences:

- **no inbound ports are open** anywhere on the network
- the origin IP is never exposed
- no certificate management — TLS terminates at Cloudflare

Routing is per-hostname, straight to a Swarm service DNS name:

```
app.example.com  →  http://<stack>_<service>:<port>
```

Traefik was tried first and abandoned. The reasoning is in
[`journey.md`](journey.md#the-traefik-detour).

> **Constraint worth knowing:** Cloudflare's terms prohibit continuous video
> streaming through their CDN, and WireGuard/UDP does not pass through the tunnel
> at all (HTTP/HTTPS only). Both are reasons the design keeps a separate VPN mesh.

## Access: two independent layers

| Layer | Mechanism | Protects |
|---|---|---|
| Outer | Cloudflare Access — e-mail OTP + authenticator MFA | Every tunnel-exposed service, before a request reaches the origin |
| Inner | The application's own session auth (bcrypt, cost 12) | The application itself |

The point of the outer layer is that nobody reaches so much as a login form without
first passing Cloudflare. The point of keeping the inner one is that the application
must not become insecure the moment it is reached by some other route — a request
arriving from inside the LAN, or a misconfigured tunnel, still has to authenticate.

**The application deliberately contains no auth-provider code.** Identity is
Cloudflare's job; the app only knows its own users.

The inner layer is designed to stand on its own, because the outer one is not
switched on yet. In practice that means: no environment-seeded admin (the first
account is created through a one-time setup wizard that seals itself), accounts
afterwards only by single-use invitation code, database-backed rate limiting and
lockout so a container restart does not reset the counters, and CSRF on every
state-changing request. The full account is in the dashboard repo's
`docs/security.md`.

**One service is deliberately not stateless: sessions.** The dashboard's session
store is process memory, because a session holds the user's derived vault key —
putting it in a cookie would hand the key to the browser, and putting it in
PostgreSQL would put it in exactly the dump the vault is designed to survive. So
that one service runs at `replicas: 1` and a restart signs everyone out. It is the
single place where the stateless rule is broken on purpose, and it is broken in
memory rather than on disk, so it still costs nothing to reschedule.

> **Status:** the tunnel and the Access policy are configured in the design and in
> the compose files, but not yet switched on in production. Until they are, the
> application's own login is the only layer. See [`journey.md`](journey.md#where-this-actually-stands).

## Management network

Every node joins a **Tailscale** mesh. Chosen over plain WireGuard for NAT traversal
and because it needs no open port; the trade-off is a dependency on a third party for
the coordination plane. Administration happens over that mesh, never over a
port-forward.

## Backups

The intended shape follows 3-2-1 — and is only partly built:

| | What | State |
|---|---|---|
| Local | Nightly `tar.gz` of the config tree and the NFS share, 7-day retention, written to a *different* node than the one holding the data | Running |
| Offsite | Cloudflare R2 (S3-compatible) | Planned |
| Database | `pg_dump` of PostgreSQL | **Not yet implemented** |
| Restore | A rehearsed restore test | **Never performed** |

That last pair is the honest gap in this build, and it is treated as one — see
[`journey.md`](journey.md#the-backup-that-isnt-one-yet).

## What is deliberately *not* here

| Not used | Why |
|---|---|
| Kubernetes | Swarm gives real clustering at a fraction of the operational cost; the complexity is not justified at this size |
| Proxmox / VMs | Configs are in git, data is on NFS and Postgres — nodes are already replaceable, so a hypervisor layer only adds overhead |
| Traefik / any reverse proxy | Evaluated and rejected; the tunnel routes directly |
| A separate identity provider | Cloudflare Access covers the outer layer without running more infrastructure |
