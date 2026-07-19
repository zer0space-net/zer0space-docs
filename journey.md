# The Journey

How this cluster ended up looking the way it does. Mostly a record of things that
did not work, because those are the parts that shaped the design.

> Addresses, domains and image paths on this page are placeholders.

---

## The Traefik detour

The plan was conventional: Cloudflare Tunnel for ingress, Traefik behind it as the
reverse proxy, services registering themselves through Docker labels. It is the
setup half the homelab internet runs.

Traefik in Swarm mode needs to read the Docker API to discover services. Mounting
the Docker socket straight into a container is handing it root on the host, so the
socket went behind `tecnativa/docker-socket-proxy` — a small filter that exposes
only the endpoints you allow.

Then nothing worked. The proxy rejected every request Traefik made.

The cause took an embarrassing amount of time to find: **Traefik's Swarm provider
was negotiating Docker API version 1.24** — a version from 2016 — regardless of the
`DOCKER_API_VERSION` environment variable set to 1.43 or 1.44. The socket proxy
advertised 1.55 with a minimum of 1.40, so it refused. Correctly.

The attempts, each its own commit:

1. Pin `DOCKER_API_VERSION=1.44` — ignored by the Swarm provider
2. Enable the `VERSION`, `INFO` and `SWARM` endpoints on the proxy so negotiation
   had something to work with — no change
3. Point Traefik at the proxy by its stack-qualified name in case it was a DNS
   problem — no change
4. Upgrade Traefik from v3.3 through v3.5, on the theory it was a fixed bug — no
   change

The only configuration that worked was mounting the Docker socket directly into
Traefik — which is exactly the thing the proxy existed to prevent. Trading a root
socket for prettier routing is a bad trade.

So Traefik was deleted, and `cloudflared` was pointed straight at the services:

```
app.example.com  →  http://<stack>_<service>:<port>
```

Cloudflare already terminates TLS, already does DNS, already has an access policy
engine. The reverse proxy had been solving a problem that was already solved one
layer up. Removing it deleted a component, a network, a socket proxy, and a class
of bug — and lost nothing.

**What stuck:** the reflex to ask *"what is this component actually for?"* before
adding it. A reverse proxy is load-bearing when you terminate your own TLS and route
your own DNS. Behind a tunnel that does both, it is mostly ceremony.

The decision is written into the repo as a note explaining why not to re-add it —
because in six months someone (me) will wonder why there is no reverse proxy.

---

## The overlay network that lied

Late-joining worker nodes were `Ready` in `docker node ls`. Services scheduled onto
them started fine. And the dashboard showed them as unreachable.

Swarm overlay networks tunnel container traffic between hosts over VXLAN. On those
particular nodes the VXLAN tunnel was silently broken — not *down*, which would have
been obvious, but broken in a way that let the control plane through while dropping
data-plane traffic. Swarm's own health checks were satisfied. Container-to-container
traffic across those hosts was not.

The symptom that made it visible: the monitoring agent published a port on the
overlay, and connections to it from other nodes returned "host unreachable" for
overlay IPs while the node itself was demonstrably alive.

Two fixes were possible: debug VXLAN across a mixed-hardware fleet, or stop
depending on it for monitoring. The second one won:

- the metrics agent switched to `mode: host` port publishing, binding directly to
  the host NIC and bypassing the overlay mesh entirely
- the backend stopped resolving agents through overlay DNS and started reading each
  node's LAN address from the Swarm API's own node list (`Status.Addr`) — which is
  authoritative, and correct even when the overlay is not

Monitoring became more reliable than it had been before the bug, because it no
longer depended on the layer most likely to fail.

**What stuck:** *"Ready" is a claim about the control plane, not about the data
plane.* And when a subsystem is both hard to fix and not essential to the job, the
cheaper move is often to remove the dependency rather than repair it.

The underlying VXLAN issue is still open. It is documented, its blast radius is
understood, and it is not pretending to be fixed.

---

## Making the Swarm forget

The first version was a normal Swarm cluster: services with volumes, volumes on
whichever node the service happened to run on, `placement.constraints` pinning each
service to *its* node so it would find its data again.

That works, and it is a lie. Seven nodes, each holding data only it can serve, is
not a cluster — it is seven single points of failure wearing a cluster costume. The
dashboard was pinned to one node by a bind-mount. Lose that node and you lose the
dashboard *and* its database.

Fixing it properly meant a rebuild in two parts.

**Files moved off the Swarm.** A dedicated storage host outside the cluster exports
one NFS share, mounted at the same path on every node. Services address
`/mnt/storage/<service>/` and no longer care which machine they are on.

**Structured data moved to a database.** The dashboard had been using SQLite — a
local file, and therefore the last thing pinning it down. Putting SQLite on NFS is
a well-known way to get corrupted data, since its locking assumes a local
filesystem. So SQLite came out and PostgreSQL went in, on the stateful host,
reached over the network.

That port was more than a driver swap. `better-sqlite3` is synchronous; `pg` is not.
Every database call in the application became `async`, which in Express 4 means
every route handler needs wrapping — Express 4 does not catch rejected promises from
an `async` handler, so an unwrapped one becomes an unhandled rejection instead of a
500. The SQL moved too: `AUTOINCREMENT` → `SERIAL`, `INSERT OR REPLACE` →
`ON CONFLICT … DO UPDATE`, `lastInsertRowid` → `RETURNING *`, `COLLATE NOCASE` →
`LOWER()`.

A one-shot migration script moved the existing rows, with a `--dry-run` mode that
runs the whole transaction and then deliberately rolls it back.

Two details that only surface when you actually run it:

- **Sequences must be reset after a migration.** Copy rows with explicit ids into
  Postgres and the sequence still points at 1 — so the next insert collides with a
  migrated row. The script calls `setval()` per table afterwards.
- **A dead connection is not a query error.** When Postgres restarts, the failure
  arrives as an `error` event on an *idle* pool client, not as a rejected query.
  Without a listener on the pool, Node treats it as an unhandled `error` event and
  kills the process. One listener turns a crash into a reconnect.

The payoff: the application now holds no state. It can be scheduled onto any node,
rescheduled mid-request, or rebuilt from an image with nothing to restore. The
pinning that remains is one socket proxy that genuinely needs a *manager's* Docker
socket — and it is pinned for that reason, documented as such, rather than because
of a volume.

**What stuck:** stateless is not a property you add at the end. It is a constraint
you accept early, and the cost of retrofitting it is roughly the cost of the port
you were avoiding.

---

## The backup that isn't one yet

The backup design is orthodox 3-2-1: a nightly `tar.gz` of the config tree and the
NFS share, 7-day retention, written to a *different* host than the one holding the
data, plus an offsite copy in object storage.

The nightly local job runs. It writes a small JSON status file per node that the
dashboard reads, so a silently-failing backup shows up as a card going stale rather
than as nothing at all — a backup that fails quietly is worse than none, because it
buys false confidence. The script uses `trap ERR` to record `failed` instead of
simply exiting.

The NFS migration nearly broke it invisibly. The job archived the config tree, which
was where service data used to live. After the move, the data was on the NFS share
and the config tree no longer contained it — the backup would have kept succeeding
while capturing progressively less of what mattered. It now archives the share as
well, from exactly one node, so two cron hosts don't duplicate gigabytes nightly.

Then the database migration opened a second hole. Users, roles, settings and the
encrypted vault now live in PostgreSQL on the stateful host — and **no backup job
covers that database.** The nightly `tar.gz` does not touch it. For those rows there
is currently no restore path at all.

And the part that is easiest to be honest about only in hindsight: **the restore has
never been tested.** Not once. There are archives on disk, a retention policy, a
status card that goes orange when a node misses a run — and no evidence that any of
it can actually bring the system back.

An untested backup is a hypothesis, not a backup. This one is written down as an
open item rather than quietly assumed to work, which is the least it deserves.
Offsite copies, a database dump, and a rehearsed restore are the next block of work.

**What stuck:** the interesting failure isn't a backup that breaks. It is a backup
that keeps reporting success while the thing worth saving quietly moves somewhere
else.

---

## Where this actually stands

A showcase is worth very little if it only shows the finished parts.

**Working:** seven-node Swarm with three managers; all persistent files on central
NFS; the application on PostgreSQL and genuinely stateless; GitOps deployment from
git through Portainer; nightly local backups with visible status; monitoring that
survives the broken overlay; a self-built dashboard with per-user encrypted
credential storage.

**Designed and committed, not yet switched on:** the Cloudflare Tunnel and the
Access policy in front of it. The compose files and the policy design exist; the
tunnel token is not deployed and the public hostnames are not configured. Until
that happens the application's own login is the only auth layer, and the
`COOKIE_SECURE` / `FORCE_HTTPS` flags stay off because there is no HTTPS in front
of it yet.

**Known broken:** the VXLAN overlay to the late-joining workers. Cluster and
monitoring both work around it; it is not fixed.

**Missing:** offsite backups, any database backup at all, and a restore test.

The gap between "architected" and "running" is the most honest thing in this
repository, and it is deliberately not smoothed over.
