# Tech Stack

Every technology in the cluster and why it is there — including the ones that were
evaluated and dropped, since those decisions are as load-bearing as the rest.

> Domains, registries and addresses are placeholders.

## Orchestration & hosts

| Technology | Purpose | Why this one |
|---|---|---|
| **Docker Swarm** | Cluster orchestration across 7 nodes | Real clustering — scheduling, overlay networking, rolling updates — without Kubernetes' operational surface. At this size, k8s costs more attention than it returns. |
| **Docker Engine** | Container runtime on every host | — |
| **Debian** | Base OS on all nodes | Stable, boring, long support. Deliberately boring: the interesting parts belong further up the stack. Fleet is intentionally mixed-version — new nodes get the newer release rather than rolling the old ones. |
| **Portainer CE** | GitOps deployment UI | Pulls compose files directly from git, so a stack's definition and its deployment cannot drift. Runs *standalone*, outside the Swarm it manages — a control plane that dies with the cluster is not a control plane. |

Bare metal throughout — no hypervisor. Configs live in git and data lives on NFS and
Postgres, so nodes are already replaceable; a VM layer would add overhead without
adding recoverability.

## Storage & data

| Technology | Purpose | Why this one |
|---|---|---|
| **NFS** | Central file storage, mounted at the same path on every Swarm node | Swarm does not replicate volumes. One share every node can reach is what makes the cluster stateless — a service finds its files wherever it starts. |
| **PostgreSQL 16** | Structured application data (users, roles, settings, encrypted vault) | Network-reachable, so the application keeps no local database file. Replaced SQLite specifically to remove the last thing pinning a service to one node. |

SQLite was the original choice and was removed. It is excellent — but it is a local
file, and putting one on NFS invites corruption, because its locking assumes a local
filesystem.

## Networking & access

| Technology | Purpose | Why this one |
|---|---|---|
| **Cloudflare Tunnel** (`cloudflared`) | Public ingress | Outbound-only connection: no open inbound ports, no port forwarding, origin IP never exposed. Routes per hostname straight to a Swarm service. |
| **Cloudflare DNS + Proxy** | DNS and edge | Free tier covers it. |
| **Cloudflare Access** | Outer auth layer — e-mail OTP + authenticator MFA | Configure once, protects every tunnel-exposed service. Nobody reaches a login form without passing it first. Keeps identity out of application code entirely. *(Designed; not yet enabled.)* |
| **Cloudflare R2** | Offsite backup target (S3-compatible) | Satisfies the offsite leg of 3-2-1 on the free tier. *(Planned.)* |
| **Tailscale** | Management VPN mesh across all nodes | NAT traversal with no open port. Chosen over plain WireGuard for that; the trade-off is depending on a third party for coordination. Administration never goes through a port-forward. |

Two constraints this stack imposes, both worth knowing before copying it:

- Cloudflare's terms prohibit **continuous video streaming** through their CDN.
- **WireGuard/UDP does not traverse the tunnel** — it carries HTTP/HTTPS only. This
  is a large part of why a separate VPN mesh exists.

## Application & CI

| Technology | Purpose | Why this one |
|---|---|---|
| **Python 3.12 + FastAPI** | The custom dashboard's backend | Replaced Node.js/Express in v4. The dashboard is mostly an aggregator of slow external APIs — nine Glances agents plus the Docker socket proxy — and `asyncio.gather` over `httpx` expresses that fan-out more directly than the callback-shaped equivalent. Type hints also make the request/response contracts checkable without a test suite. |
| **uvicorn** | ASGI server, **one worker** | Not a tuning decision: the session store is process memory and holds each user's derived vault key, so a second worker would serve requests that cannot see the session the first one created. |
| **Vanilla JS frontend** | Dashboard UI — no framework, no build step | Editing a file and reloading the browser is the entire dev loop. At this size a bundler would be pure overhead, and the no-build property means the served files are the files in git. |
| **Jinja2** | Server-rendered page shells | Each page ships as HTML with its own `<title>` and content; only the dashboard itself behaves as an SPA. |
| **`asyncpg`** | PostgreSQL driver | Plain parameterised SQL, no ORM. Native `$1` placeholders, and it returns real `datetime` objects instead of strings. |
| **`bcrypt`** | Password hashing (cost 12) | Run in a worker thread — a cost-12 round is ~250 ms of CPU, and doing it inline on a single-worker ASGI server is a denial of service anyone can trigger by holding the sign-in button down. |
| **`cryptography` + `hashlib`** | Vault encryption — PBKDF2-HMAC-SHA256 (600k iterations) + AES-256-GCM | The key is derived from the user's password at login and lives only in the server-side session, so a database dump alone cannot decrypt it. The stored format is byte-identical to what the Node implementation wrote, so the rewrite needed no data migration. |
| **`itsdangerous`** | Signs the session-id cookie | The cookie carries *only* a signed id. Starlette's own session middleware serialises the whole session into the cookie, which would hand the vault key to the browser — so it is deliberately not used. |
| **GitHub Actions** | Builds and publishes the dashboard image on push | Image goes to a container registry; Portainer pulls and redeploys. |
| **Glances** | Per-node host metrics, `mode: global` | One agent per node automatically, including nodes that join later. Publishes on a host port rather than the overlay — deliberately, after the overlay proved unreliable. |
| **`docker-socket-proxy`** | Read-only, endpoint-filtered access to the Docker API | The dashboard needs Swarm status. Mounting the raw socket into a container is root on the host; this exposes only `/nodes`, `/services` and `/tasks`, and nothing else. |
| **Stirling-PDF** | Self-hosted PDF toolkit | Stateless — conversions happen in container temp space, so it needs no storage at all. |

## Evaluated and rejected

| Technology | Verdict |
|---|---|
| **Traefik** | Removed after several attempts. Its Swarm provider negotiated Docker API 1.24 regardless of configuration, which the socket proxy correctly refused. The only working fix was mounting the raw Docker socket — trading a root socket for routing convenience. Cloudflare already does TLS, DNS and access control, so the reverse proxy was solving an already-solved problem. See [`journey.md`](journey.md#the-traefik-detour). |
| **Kubernetes** | Rejected up front. Swarm covers the actual requirements; k8s complexity is not justified at nine machines. |
| **Proxmox / virtualisation** | Rejected. Nodes are already disposable via GitOps and network storage. |
| **A self-hosted identity provider** | Dropped as a priority once Cloudflare Access covered the outer auth layer — one less service to run and secure. |
| **SQLite** | Replaced by PostgreSQL. Not a flaw in SQLite: a local file is incompatible with a stateless cluster, and NFS is the wrong filesystem for it. |
| **Node.js + Express** | Replaced by Python/FastAPI in the v4 dashboard rewrite. It worked; the async fan-out across nine metric endpoints and the ergonomics of typed request handling were the reasons to move, not a failure. |
| **A frontend framework for the dashboard** | Rejected again at the v4 rewrite. The UI is a few hundred lines of DOM building; a framework would add a build step, and with it the gap between what is in git and what is served. |
| **Starlette's `SessionMiddleware`** | Rejected. It stores the session inside the cookie, and the session holds the vault key. A database-backed store is out for the mirror-image reason. |

## Reference examples

Two generic, copy-pasteable patterns extracted from this build:

- [`examples/service-behind-tunnel.yml`](examples/service-behind-tunnel.yml) — putting
  a service behind a Cloudflare Tunnel with no open ports
- [`examples/nfs-volume-pattern.yml`](examples/nfs-volume-pattern.yml) — persistent
  storage for a stateless Swarm, both the simple and the fail-loudly variant
